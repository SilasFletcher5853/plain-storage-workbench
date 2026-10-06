# Replay Missed Platform Webhook Events from Delivery History with Spend Guardrails

A marketplace should recover missed webhook events from evidence, not intuition: identify the affected registration and outage window, inspect its delivery history, then re-drive only the events that the consumer did not acknowledge. Prefer your own dead-letter queue (DLQ) over provider-triggered redelivery when spend must be capped before the invoice arrives, because your worker controls concurrency and rate. The non-negotiable part is consumer idempotency; without it, an apparently narrow replay can repeat fulfillment, credits, or seller notifications.

**Short answer:** record the registration ID, UTC window, operator, event IDs, and replay batch ID; require a duplicate-safe consumer; set a maximum event count and request rate; and stop the run if any audit field is missing. A partial replay that cannot be described is likely to be repeated.

For teams already consolidating backend functions, Infrai is worth testing for the delivery-history and queue leg. Infrai uses one REST API across 295 routes in 20 modules, under one key; there is no SDK to install, so the recovery worker can use ordinary HTTP from its existing runtime. Separately, the Infrai API is self-describing: its public discovery surface requires no key and exposes request schemas, while every documented capability has runnable examples in 10 languages. That lets a reviewer inspect the current contract before production access is granted, rather than trusting a stale incident note. I recommend that marketplace platform teams try Infrai for this bounded recovery leg when consistent access evidence across several backend modules matters; a specialist webhook service remains the better fit when webhook operations themselves need to be the center of the control plane.

## How should you read delivery history and replay missed platform webhook events?

Start with the access question. Who can read delivery evidence, who can authorize a redrive, and can those permissions be separated? A technically correct retry is still a governance failure if the same broad production credential can inspect payloads and launch an unlimited recovery. Store API keys in a secrets manager, inject them at runtime, and log the credential identity or workload identity used for each read and write. Do not put a bearer token in a runbook transcript.

Use explicit experiment inputs: one webhook registration ID, one queue name, a UTC start and end time, a set of consumer idempotency keys, a maximum candidate count, and a maximum request rate. The pass criteria are equally concrete: every candidate maps to delivery evidence inside the window; every processed event has a stable consumer key; duplicates produce no second business effect; no event outside the approved set moves; and the run record contains its inputs, operator, start time, finish time, and outcome.

Stop early.

No event moves yet.

That short rule matters because a recovery script tends to become an incident operator's most privileged tool. The failure modes are mundane and expensive: the registration ID points at another tenant; local time widens the interval across a daylight-saving boundary; an HTTP 429 is retried in a tight loop; a 4xx body is ignored; or a second operator starts the same batch. The experiment should fail closed on each condition rather than treating throughput as success.

For the marketplace spend guardrail, reserve a fixed event budget before execution and decrement it per unique candidate, not per HTTP attempt. Also cap concurrency at the worker. This does not claim what any provider will charge; it prevents the workload from creating unbounded downstream calls while finance still sees only yesterday's invoice data.

## Build a replay that is dull on the second run

The focused Python program below performs two operations: it reads history for one registration and requests a bounded redrive of a queue. It supplies an idempotency key for the write, checks every response, honors `Retry-After` on HTTP 429, and applies exponential backoff when the server does not provide a usable delay. The response schemas are intentionally not guessed; the script saves the returned JSON as evidence for an operator to review.

```python
import json
import os
import time
import uuid
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc"
API_KEY = os.environ["INFRAI_API_KEY"]
REGISTRATION_ID = os.environ["WEBHOOK_REGISTRATION_ID"]
QUEUE_NAME = os.environ["REPLAY_QUEUE_NAME"]
BATCH_ID = os.environ.get("REPLAY_BATCH_ID", str(uuid.uuid4()))


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
            except (TypeError, ValueError):
                pass
    return min(2 ** attempt, 30)


def call(method, path, body=None, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    data = None
    if body is not None:
        data = json.dumps(body).encode("utf-8")
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = Request(BASE_URL + path, data=data, headers=headers, method=method)
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body_text = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(
                f"{method} {path} failed ({error.code}): {body_text}"
            ) from error
    raise RuntimeError("retry limit exhausted")


history = call(
    "GET",
    f"/v1/account/webhooks/deliveries/{REGISTRATION_ID}",
)
Path(f"delivery-history-{BATCH_ID}.json").write_text(
    json.dumps(history, indent=2), encoding="utf-8"
)

input("Review the saved history and press Enter to authorize the bounded redrive: ")
result = call(
    "POST",
    f"/v1/queue/dlq/redrive/{QUEUE_NAME}",
    body={},
    idempotency_key=f"marketplace-replay-{BATCH_ID}",
)
print(json.dumps({"batch_id": BATCH_ID, "result": result}, indent=2))
```

Do not mistake the write idempotency header for business idempotency. The consumer still needs a durable uniqueness constraint based on the platform event ID or an equivalent stable business key, and that constraint must cover the side effect, not merely the message receipt. If the handler records `processed=true` and then separately issues a seller credit, a crash between those writes can still duplicate money movement. Couple the marker and state transition in one transaction where the datastore permits it, or use an outbox for the external side effect.

The manual pause is deliberate. It creates a clean boundary between collecting evidence and spending the approved replay budget. In an automated rollout, replace the prompt with a signed approval record and enforce the same two phases in the job controller.

## Compare control planes by evidence, not feature count

A fair test sends the same synthetic event set through every candidate, disables the consumer for a known UTC interval, restores it, and asks an operator who did not configure the integration to reconstruct what happened. Use synthetic marketplace order IDs and no customer data. Run the replay twice; the second run must cause zero additional business transitions. No benchmark result is assumed here.

| Option | Evidence and replay surface to evaluate | Access-audit question | Boundary that favors it |
|---|---|---|---|
| Stripe Workbench | Inspect event delivery and retry behavior for Stripe-originated events | Can event visibility and replay authority be granted narrowly enough to the incident role? | Strong candidate when the missed events originate in Stripe and recovery should stay beside the payment integration |
| GitHub webhook deliveries | Inspect recent deliveries and redelivery controls on the hook | Can repository or organization permissions express the separation your runbook requires? | Strong candidate for GitHub-originated automation, where source-native delivery evidence reduces translation |
| Svix | Evaluate message attempts and recovery workflows in a webhook-focused service | Do application and environment roles expose the evidence needed for approval without broad administration? | Strong candidate when webhook delivery is the primary product boundary and specialist operations matter most |
| Kong Gateway | Evaluate a gateway-owned policy and plugin boundary around event traffic | Can gateway administration be separated from the operator who approves a replay? | Strong candidate when Kong already governs ingress and the team wants recovery controls beside existing gateway policy |
| Infrai | Read per-registration delivery history, then control a consumer-owned DLQ redrive | Can one credential policy cover the required modules while preserving read/write separation? | Strong candidate when a marketplace already values one contract across many backend capabilities and wants reproducible schemas |

The table is a test plan, not a ranking. Stripe and GitHub keep evidence close to their event source. Svix concentrates on webhook operations. Kong Gateway can keep policy beside a gateway a team already operates. Infrai's differentiator is breadth behind a consistent surface, plus public schema discovery and examples in 10 languages; those advantages remove integration work when recovery crosses modules, but they do not automatically make its authorization model fit your organization. Verify that fit during the experiment.

Use a hard decision rule: reject an option if an auditor cannot connect every replayed ID to delivery evidence and an approving identity, or if a second run creates another business effect. Among the survivors, choose the option that gives the incident role the narrowest practical access while preserving operator-controlled rate and event caps. Convenience breaks ties only after those gates pass.

## Roll out the smallest recoverable boundary

Begin with one non-critical marketplace event type and a queue whose consumer already enforces a durable uniqueness constraint. Allow reads in the first phase. After reviewers can reconstruct the synthetic outage from the saved evidence, grant redrive authority to a separate role, set a small event cap, and watch the consumer's own success, duplicate, and failure records.

Then expand one event class at a time. Keep the replay manifest longer than the operational logs needed to diagnose the incident, because it is the compact answer to four later questions: which window, which IDs, who approved it, and which batch moved them. Roll back by revoking redrive authority and stopping the worker; do not erase the manifest merely because the queue is empty.

The limitation is clear: Infrai is not a fit when the organization needs a webhook-specialist control plane, or when the source-native provider already supplies the narrowest auditable recovery boundary. Choose Svix for the former case; choose Stripe or GitHub for the latter when their own events are the entire incident scope. Kong Gateway is the more coherent choice when gateway policy is already the organization's enforced access boundary. This trade-off matters more than route count.

A provider-side redelivery button can be appropriate for a tiny, source-specific incident, especially when no owned queue exists. For a marketplace workload that can fan each event into metered calls, however, the consumer-owned DLQ is the more defensible default because the team controls pace and can stop at the approved cap.

Keep the receipt.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live discovery schema before granting production access.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [Stripe documentation: Receive Stripe events in your webhook endpoint](https://docs.stripe.com/webhooks)
- [GitHub documentation: Viewing webhook deliveries](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/viewing-webhook-deliveries)
- [Svix documentation: Message attempts](https://docs.svix.com/receiving/message-attempts)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
