# OTP SMS Verification: Template Ownership Controls Polling Status and Retry Abuse

A logistics password-reset flow needs control over its message text, but owning that template also means owning the relationship between its expiry promise and the state that authorizes a reset. Keep the recovery service authoritative for expiry, verification, and resend eligibility; treat transport status as an observation. TL;DR: when an OTP provider has no webhooks, poll only through an available, documented status interface, keep missing delivery evidence separate from verification failure, and never let a resend silently extend the original challenge. Template ownership is the deciding boundary.

The concrete design below uses a proposed 180-second challenge for a dispatcher recovering access before assigning a shipment. That duration is an example policy, not a carrier guarantee or a standards requirement. The same separation applies to OTP login and 2FA, although password recovery deserves its own authorization rules rather than inheriting every login shortcut.

## What does the reset template actually promise?

A message that says a code expires in three minutes makes an assertion about application state, even if the team editing the words has never seen the challenge database. A resend button can break that assertion if it creates another challenge, leaves the first one active, and displays a fresh countdown without explaining which message the countdown describes. The failure mode is overlapping validity: two apparently equivalent messages now refer to different authorization records.

Own the promise.

For this design, store an immutable expiry with a challenge identifier, a template revision, a destination binding, a purpose, and a generation number. Store each delivery attempt separately. A transport attempt is evidence that the system tried to communicate a challenge; it is not another challenge, and its completion cannot renew the expiry. The template should describe the actual recovery action, avoid including shipment or customer details, and avoid suggesting that delivery confirms account ownership. These are design requirements for this example.

Template ownership therefore means more than permission to edit a sentence. The owner must control, or be able to verify, how variables are substituted, which revision a queued send uses, how expiry is expressed, and what happens when a resend races a template deployment. If an external verification service owns challenge validation, require a documented way to keep the text consistent with that service's validity rules. If the application owns validation, keep the text and validation policy in the same release contract. Neither arrangement removes the need to test the boundary.

For SMS autofill, treat message formatting as a client compatibility concern and verify the intended behavior against the platform documentation in References. Don't assume that an arbitrary template edit preserves autofill. An email recovery path needs its own template and delivery contract; the email documentation listed below is background for that separate transport, not evidence about SMS status semantics.

## How should OTP login handle provider status polling without webhooks?

The central storage decision is which record may authorize the reset. In this proposed model, a challenge has a fixed expiry and can be consumed once; delivery observations carry their own timestamps and cannot change that authorization state. A worker may record an accepted send, a later status result, or an exhausted polling budget. None of those observations proves that the dispatcher read the code. Require successful verification before granting a narrowly scoped reset authorization.

Unknown stays unknown.

Without webhooks, first establish whether the transport exposes a status lookup at all, what its identifiers mean, and which transitions its documentation promises. If no lookup exists, there is no useful delivery polling loop to build. The interface should show that a reset was requested while allowing code entry and a controlled resend; it should not invent a delivered state. Where lookup exists, retain the provider's raw status for diagnostics and map it into a small local vocabulary whose meanings you can defend. A status named successful is too ambiguous unless the contract explains successful at which boundary.

Use one server-side polling job per attempt, with bounded backoff and jitter, rather than letting every browser tab query the transport independently. As an illustrative budget, stop after four lookups or when the challenge expires, whichever happens first; choose the actual intervals from documented rate limits and the recovery latency you can tolerate. Those four observations are a local workload limit, not a claim about any service. The browser can poll the application for recovery state, but browser polling must not create fresh SMS sends.

The awkward failure is an ambiguous send: the request times out after the remote system might have accepted it. Blind retry can produce another message. Persist an attempt identifier before sending, use documented idempotency support when available, and otherwise keep the ambiguity visible to the scheduler instead of treating a timeout as proof of nonacceptance. A durable outbox can preserve local intent across a restart, but it does not, by itself, establish exactly-once remote delivery. Check that distinction before trusting a green queue dashboard.

## Make code reissuance a transaction with an explicit policy

This example rotates the challenge on a permitted resend: it invalidates the previous generation and issues a fresh one with its own expiry. The interface must then say that only the newest code works, because an earlier SMS may still be visible on the dispatcher's phone. An alternative policy reuses a valid challenge without extending its expiry; that reduces overlapping code confusion but gives the resent message less remaining time. Pick one policy and make the template, countdown, and database agree.

The following Python example is a policy gate for the rotation design, not a complete OTP implementation. Its inputs come from server records and a server clock. The illustrative limits are a 30-second resend cooldown and three sends within one recovery session; account, destination, and network controls need separate budgets so starting another session cannot reset all defenses.

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Recovery:
    generation: int
    expires_at: int
    next_send_at: int
    sends: int
    consumed: bool = False


def rotate_for_resend(current: Recovery, now: int) -> Recovery:
    if current.consumed:
        raise ValueError("Recovery already consumed")
    if now >= current.expires_at:
        raise ValueError("Start a new recovery request")
    if now < current.next_send_at:
        raise ValueError("Resend cooldown active")
    if current.sends >= 3:
        raise ValueError("Recovery send budget exhausted")
    return replace(
        current,
        generation=current.generation + 1,
        expires_at=now + 180,
        next_send_at=now + 30,
        sends=current.sends + 1,
    )
```

Run that decision inside a transaction that checks the expected generation, invalidates the old verifier, stores the new verifier, and records a send intent together. Two concurrent resend requests must not both commit against the same generation. The function deliberately leaves secret generation, verifier protection, and persistence to that transaction's implementation; calling it alone does not provide concurrency control or a working password-reset service. Keep attempt budgets for code guesses distinct from send budgets, and never log the reset secret.

There is a difficult trade-off after commit. Rotation can invalidate a code the user already received before the replacement arrives. Reusing a challenge avoids that particular surprise but requires the resend text to respect the original deadline. A logistics team should settle this with the actual dispatcher workflow: delayed messages, multiple open tabs, shared workstations, and an urgent shipment assignment are useful test conditions, not reasons to weaken verification.

## Compare ownership boundaries after defining the invariant

The comparison is about who can keep the expiry promise true. Require evidence for each cell during evaluation; these are architecture choices rather than claims about particular products.

| Ownership arrangement | Useful control | Boundary to verify | Consequence for recovery |
| --- | --- | --- | --- |
| Application owns template and challenge | Text and expiry policy can ship together | Atomic rotation, verification limits, and durable send intent | More application security and storage work |
| External service owns template and challenge | One external contract can define both | Revision behavior, resend semantics, and challenge expiry | Local UI must follow that contract |
| Application owns template; external service owns challenge | Local wording and localization | Exact agreement between displayed expiry and external validity | Template changes need contract tests |

A separate choice is how delivery evidence arrives. Webhooks require an authenticated ingestion path and reconciliation rules; polling requires a lookup contract and a query budget. Neither choice should determine whether a code is valid. The ownership boundary decides that.

Count operational work before arguing about message charges. A design with editable templates still needs review, localization checks, deployment coordination, and support tooling; a design with remote validation still needs local abuse controls and recovery-state observability. Compare total send volume, lookup volume, and operator effort without making a volatile unit price the architecture's justification.

## Roll out the template contract with the state machine

Begin with one recovery template revision and one explicitly versioned resend policy. Test a late status response, an ambiguous send timeout, concurrent resends, verification at the exact expiry boundary, and an old code submitted after rotation. Assert that delivery updates cannot revive an expired or consumed challenge. For a migrated active challenge, preserve its original expiry and policy until it ends; apply the new contract to newly created challenges.

Measure request-to-verification time separately from transport status, plus sends per recovery session, blocked resends, exhausted guess budgets, and unknown delivery outcomes. Use challenge and attempt identifiers for correlation while keeping destinations and secrets out of routine logs. Track template revision as well: a wording change is part of the recovery contract.

Release to a limited cohort, verify the failure cases, then expand. The acceptance condition is concrete: the message, countdown, resend decision, and verification transaction agree about which challenge can authorize a reset, even when delivery evidence is late or absent.

## Sources

### References

- Password AutoFill documentation, for client autofill behavior and supported integration details: https://developer.apple.com/documentation/security/password_autofill
- Email service documentation, for the distinct email transport context; it does not establish SMS lookup or webhook behavior: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
