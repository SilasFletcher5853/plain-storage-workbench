# Node.js Recipient State Explained (Transactional Email Polling and SMS Fallback)

Treat recipient validity as durable media-platform data, then let notification workers consult it before email and update it only from conclusive bounce evidence. **TL;DR:** a Node.js event pipeline should submit email first, poll unresolved submissions within a fixed window, suppress an address only after a permanent invalid-recipient result, and use SMS only when an explicit escalation rule allows it. The deciding constraint is integration effort: one shared recipient-state boundary is harder to add than a timer, but far easier to reason about than bounce logic copied into every publishing workflow.

This is an architecture decision record for contributor notices, licensing updates, and publication events. It does not treat a transport's acceptance response as delivery, a late status as failure, or SMS as proof that a person saw anything. Those distinctions sound fussy until an editor corrects an address while an old poll job is still running.

## How should Node.js event notifications poll transactional email before SMS fallback?

The first invariant is about identity: suppression belongs to a normalized email address and its evidence, while an attempt belongs to one notification and one channel. A photographer can change an address; a story can trigger several notices; a polling response can be repeated. Those are different facts and need different keys.

The second invariant is stricter. Only a permanent invalid-recipient classification may automatically suppress an address. Submission acceptance, an unresolved poll, a temporary failure, and a configured deadline are not evidence that the mailbox is invalid. **Unknown stays unknown.**

No inference.

The remaining boundaries are operational rather than cosmetic:

- reserve no more than one email attempt and one SMS attempt for a notification;
- attach every observation to the address version and attempt that produced it;
- deduplicate observations by a stable evidence key;
- stop polling at a declared deadline without manufacturing a delivery result;
- never let an old bounce suppress a newly corrected address.

That last failure mode deserves attention. Suppose `alex@example.invalid` is corrected after an email submission but before its permanent bounce is observed. If the poller updates a mutable contributor row by contributor ID alone, it may suppress the replacement address. An evidence record tied to the original normalized address cannot make that mistake. This is why the schema needs 2 uniqueness constraints rather than one overloaded status field: the evidence key answers whether the same observation was already processed, while the notification-and-channel key answers whether contact was already attempted. Neither substitutes for the address-version comparison. Small key, large consequence.

DMARC does not close this gap. RFC 7489 specifies domain-level message authentication policy and reporting; it does not establish application-level receipt for one media notice. Likewise, an SMS fallback is a notification channel, not an authenticator decision. NIST SP 800-63B describes use of the public switched telephone network as a restricted authenticator, so notification policy and authentication policy should remain separate.

## Choose the smallest integration that preserves recipient truth

The useful comparison is not feature count. It is where recipient truth lives, how much reconciliation code the team owns, and what happens when status evidence never arrives.

| Integration shape | Integration effort | Recipient-state risk | Failure boundary | Appropriate use |
|---|---:|---|---|---|
| Fixed delay, then SMS | Low | Silence can be mistaken for invalidity if policy is careless | Late email evidence can cause duplicate contact | Time-sensitive notices where both channels are explicitly acceptable |
| Polling in each workflow | Moderate at first, high over time | Mapping and suppression rules drift between jobs | Each workflow owns retries and deadlines | One isolated, short-lived workflow |
| Shared recipient-state service with bounded polling | Moderate | One mapping boundary can be reviewed and tested | Scheduler failure leaves attempts unresolved, not falsely failed | Several media workflows sharing contacts |
| Callback ingestion plus bounded polling | Highest | Central rules remain consistent | Two evidence paths require deduplication | High-consequence notices where missing callbacks justify repair polling |

For a media platform with several producers of contributor mail, the third shape is the default decision here. It centralizes the dangerous operation, suppression, without requiring two inbound and outbound evidence paths on day one. Polling remains bounded repair work, not a perpetual status mirror.

**The datastore is the enforcement point.** A unique reservation on `(notification_id, channel)` prevents concurrent workers from creating duplicate channel attempts. A separate uniqueness rule on the evidence key prevents a repeated poll result from being applied twice. Recipient state stores the normalized address, classification, evidence reference, and effective time; it does not need the editorial message body.

The limitations are plain. This design needs a durable job runner, transactional writes, retention rules for contact data, and a mapping from external status vocabulary into a deliberately small internal vocabulary. It can't promise exactly-once remote delivery, because a process can fail after a transport accepts a submission but before the local identifier is recorded. Stable submission keys help only where the transport contract honors them; ambiguity must still be recorded and reconciled. It is not suitable for a team that can't operate durable polling or investigate an unresolved queue; callback-only handling with an honest `UNKNOWN` state is the lower-effort choice when missing evidence is acceptable.

## Put the policy on the critical path

Although Node.js owns the surrounding event application in this scenario, the policy is shown in Python to keep it independent of an SDK and make the state transitions easy to inspect. The Node.js worker can call the same repository operations through an internal boundary. Transport adapters map their external states before this function runs; an unmapped value becomes `UNKNOWN`.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class Outcome(str, Enum):
    DELIVERED = "delivered"
    TEMPORARY_FAILURE = "temporary_failure"
    PERMANENT_INVALID_RECIPIENT = "permanent_invalid_recipient"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class PollResult:
    attempt_id: str
    address_version: str
    outcome: Outcome
    evidence_key: str
    observed_at: datetime


def reconcile(notification_id, repository, email_transport, sms_transport, now=None):
    """Advance one notification without inferring facts from elapsed time."""
    now = now or datetime.now(timezone.utc)
    notice = repository.get_notification(notification_id)
    email_attempt = repository.reserve_email_once(
        notification_id=notification_id,
        address_version=notice.address_version,
    )

    if email_attempt.was_created:
        if repository.is_suppressed(notice.normalized_email):
            repository.mark_skipped(email_attempt.id, "recipient_suppressed")
            return
        submission = email_transport.submit(
            recipient=notice.normalized_email,
            payload=notice.email_payload,
            idempotency_key=email_attempt.id,
        )
        repository.record_submission(email_attempt.id, submission.external_id)
        return

    if not repository.poll_is_due(email_attempt.id, now):
        return

    external = email_transport.fetch_status(email_attempt.external_id)
    result = PollResult(
        attempt_id=email_attempt.id,
        address_version=email_attempt.address_version,
        outcome=external.normalized_outcome,
        evidence_key=external.evidence_key,
        observed_at=now,
    )

    with repository.transaction():
        inserted = repository.append_observation_once(result)
        if not inserted:
            return

        if result.outcome is Outcome.PERMANENT_INVALID_RECIPIENT:
            repository.suppress_if_current_address(
                address_version=result.address_version,
                evidence_key=result.evidence_key,
            )
            sms_attempt = repository.reserve_sms_once(notification_id)
        else:
            sms_attempt = None

    if sms_attempt is not None:
        submission = sms_transport.submit(
            recipient=notice.phone_number,
            payload=notice.sms_payload,
            idempotency_key=sms_attempt.id,
        )
        repository.record_submission(sms_attempt.id, submission.external_id)
```

The `suppress_if_current_address` compare-and-set is the important line. It prevents evidence for an obsolete address version from contaminating the corrected recipient record. The transaction also places suppression and the SMS reservation on one side of a crash boundary; the remote SMS submission remains outside the transaction because a database cannot atomically commit another service's acceptance.

This example intentionally omits arbitrary retry loops. Poll cadence, maximum attempts, and the final deadline belong to persisted job policy, and exhaustion produces `UNKNOWN`. Callback authentication, schema validation, and correlation-ID lookup belong at an ingestion boundary if callbacks are later added.

## Test disorder, then operate the backlog

Happy-path HTTP mocks prove little. Tests should permute events: two workers reserve email concurrently; the address changes before a permanent bounce appears; the same polling evidence is read twice; a temporary failure is followed by delivery; the deadline passes without a terminal result; and an SMS reservation exists when another evaluator runs. The assertions are compact: no obsolete address becomes suppressed, no notification gains two active attempts for one channel, and no non-permanent outcome creates suppression.

Observability should describe work and age, not imply human receipt. Track unresolved attempts by age, polls attempted, evidence deduplication, suppression changes by reason, address-version mismatches, evaluator failures, and SMS reservations. A delivery ratio can describe a trend. It cannot prove that a contributor read a licensing notice.

Deployment needs one compatibility rule: add new normalized outcomes before adapters emit them, and treat unknown external values as `UNKNOWN` during mixed-version rollout. Keep policy versions on notifications so a replay does not silently apply a newer escalation rule to old intent. Retain contact identifiers, outcome timestamps, and evidence references only for the period justified by the workflow; diagnostic status usually does not require retaining message bodies.

## Why reject timer-only fallback?

A timer-only design says: submit email, wait, then send SMS unless delivery has appeared. I reject it for bounce-driven recipient hygiene because elapsed time says nothing about address validity, and a delayed observation can turn routine latency into both duplicate contact and a poisoned suppression list.

It still has a valid use case. If the documented business rule requires two-channel outreach by a deadline, duplicate contact is acceptable, and recipient validity is governed elsewhere, a fixed delay has less integration surface and fewer retained status records. The rejection is conditional, not universal.

For shared media contacts, keep the more conservative boundary: permanent evidence may change recipient state; absence of evidence may only keep a job unresolved. That rule costs a scheduler and a small state model. It also makes the system explainable when the network, workers, and status source disagree.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
