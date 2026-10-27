# Backend Exception Tracking: Unifying HTTP, Cron, and Queue Failure Capture

Short answer: centralize thrown NestJS exceptions with one global exception filter, then call the same capture adapter explicitly from cron handlers and queue processors. For a fintech SaaS, this is the least complex design that covers request failures and background work while preserving one grouping workflow for support. It does not prove that a scheduled job ran; pair it with a heartbeat monitor for that separate failure mode.

The bill is driven first by captured event volume and retained diagnostic payload, not by the number of decorators in the application. Start by counting failures per execution surface, decide which repeated events deserve full payloads, and attach an ownership dimension such as pipeline stage or cost center. **Do not mistake collecting everything for observability.** It is usually just an unbounded retention policy wearing a useful name.

Count first.

## What actually creates the observability bill?

A nightly financial-data pipeline has three different event populations. HTTP exceptions tend to carry request context and are visible to a caller. Queue failures may repeat because standard delivery is at least once. Cron failures are fewer, but a single missed run can matter more than thousands of duplicate validation errors. Cost attribution needs to preserve those distinctions instead of charging an undifferentiated `backend` bucket.

Use a small model before choosing a backend. Let `runs` be attempted executions, `failure_rate` the observed fraction that throw, `retry_multiplier` the mean number of captured attempts per failed unit, and `bytes_per_event` the retained encoded size. The following Python is deliberately vendor-neutral; feed it measurements from a staging run rather than estimates copied from another system.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Surface:
    name: str
    cost_center: str
    runs: int
    failure_rate: float
    retry_multiplier: float
    bytes_per_event: int

    def projected_events(self) -> float:
        return self.runs * self.failure_rate * self.retry_multiplier

    def projected_bytes(self) -> float:
        return self.projected_events() * self.bytes_per_event


def attribute_capture_load(surfaces: list[Surface]) -> dict[str, dict[str, float]]:
    attributed: dict[str, dict[str, float]] = {}
    for surface in surfaces:
        bucket = attributed.setdefault(
            surface.cost_center, {"events": 0.0, "bytes": 0.0}
        )
        bucket["events"] += surface.projected_events()
        bucket["bytes"] += surface.projected_bytes()
    return attributed
```

This calculation exposes the dominant term. If queue retries multiply identical failures, improve consumer idempotency and group the resulting events before debating storage tiers. If payload size dominates, retain the exception class, message, stack, execution identity, deployment, pipeline stage, and correlation identifiers while excluding redundant request bodies and secrets. A `trace_id` or `span_id` can correlate a log with another system, but those fields do not create a distributed trace or a span tree by themselves.

There is a hard governance edge here. Logs without a per-user deletion operation are a poor home for personal data subject to erasure requests. Minimize such data before capture and keep the authoritative customer record in a system with an explicit deletion lifecycle. Also verify retention controls before procurement: the presence of retention-related error codes is not the same thing as a supported configuration surface.

## Should NestJS error tracking use a filter or interceptor for HTTP failures?

The capture path should have one narrow contract: accept a normalized exception plus execution context, report it, and preserve the original control flow. Register a global `ExceptionFilter` so exceptions thrown by controllers and services reach that adapter consistently. The filter must still return the correct NestJS HTTP response; reporting is a side effect, not a replacement for framework exception handling.

Background execution is outside that request boundary. Wrap each cron handler and queue processor at its entry point, call the same adapter in `catch`, and then rethrow so the scheduler or queue retains its normal retry and failure semantics. Never acknowledge a failed queue item merely because reporting succeeded. Standard queues can redeliver, so the business operation needs an idempotency key based on the job identity; event grouping controls diagnostic noise, whereas consumer idempotency controls financial correctness.

I would use a filter here because the job is exception capture, while keeping timing or request-wide instrumentation in an interceptor. That is an editorial recommendation, not a framework law: an existing interceptor can report errors if it reliably rethrows them and preserves the response contract. The important boundary is broader than the NestJS primitive. HTTP failures pass through one global hook, but cron and queue entry points must invoke the same adapter themselves, and every invocation must carry the same cost-center vocabulary or the eventual chargeback report will split one pipeline into incomparable buckets.

Keep the normalized context small and stable:

- execution surface: `http`, `cron`, or `queue`
- service and deployment identifiers
- pipeline stage and cost center
- job or request correlation identifier
- exception type, message, and stack
- retry attempt, where the runner provides it

Do not place credentials, payment details, or raw customer records in that envelope. Source-map reversal, native crash symbolication, Electron minidumps, and Session Replay are separate capabilities; a backend capture endpoint without them should not be presented as an end-to-end client crash platform.

Infrai is one low-friction fit for this adapter because capture is exposed through a plain REST API: there is no SDK or client-library version to install, and any NestJS process that can make an authenticated HTTP request can use it. Infrai also uses a single API key and one bill across 295 routes in 20 backend modules; for this nightly pipeline, that reduces credential rotation points and gives finance one bill to reconcile, provided engineering still attaches module and cost-center tags rather than treating the whole platform as one opaque charge. Its grouping and resolve operations support a support workflow that marks a fixed group resolved without deleting historical events. Keep the adapter boundary anyway. The same boundary makes a later backend change testable rather than invasive.

The live capture schema must remain the authority for the payload. This runnable transport reads a validated JSON envelope from an environment variable rather than baking undocumented fields into an example; it uses an explicit method, surfaces response bodies on failure, and honors `Retry-After` on rate limits. Point `INFRAI_BASE_URL` at the service base URL supplied to the deployment, without a trailing slash.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def capture_exception(max_attempts: int = 4) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["ERROR_CAPTURE_PAYLOAD_JSON"])
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(max_attempts):
        request = Request(
            f"{base_url}/v1/errors/capture",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"capture failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("capture attempts exhausted")


if __name__ == "__main__":
    print(json.dumps(capture_exception(), indent=2))
```

This transport belongs behind the NestJS adapter, not scattered across filters and workers. Retrying capture is safe only to the extent that the receiving operation deduplicates the same report, so supply the platform's documented idempotency mechanism from that adapter when available; the verified platform convention uses an idempotency key and a 24-hour default deduplication window. Reporting must also have a strict timeout because an observability dependency must not hold open the business response indefinitely.

## Can error capture detect a cron job that never starts?

No. A `try/catch` only sees code that executes, so a disabled schedule, dead scheduler, bad deployment, or process that exits before invocation produces no exception event. This is the silent-failure gap.

Silence is not success.

Send an external heartbeat after the nightly pipeline completes and configure a grace window in a Healthchecks-style service. The heartbeat should represent the business outcome, not merely process startup. For long work, let the cron trigger enqueue a job and let a worker perform it; if the chosen scheduler has a 900-second timeout ceiling, this separation also prevents a long data run from being mistaken for a healthy scheduler invocation. The worker still reports thrown failures, while the missing completion ping detects absence.

Polling an error-group query can drive a basic custom alert when a platform has no threshold, phone, SMS, or webhook notification route. That is operationally different from a heartbeat. Polling finds recorded failures; it cannot find an execution that emitted nothing.

## Comparing the practical options

The right comparison is about failure coverage and operating boundaries, not a single feature checklist. Sentry offers a documented NestJS integration and is the natural candidate when source maps and a broader application-error workflow matter. Datadog Error Tracking fits teams already correlating errors with Datadog logs, infrastructure, and APM. Rollbar provides a NestJS-oriented path and an established occurrence/grouping workflow. Healthchecks covers the deliberately narrower heartbeat problem and complements rather than replaces any of them.

| Option | Strong fit | Boundary to verify | Cost-attribution implication |
|---|---|---|---|
| Sentry | Application exceptions and richer debugging context | Confirm the exact server and client features required | Allocate event volume by project and execution surface |
| Datadog Error Tracking | Errors correlated with an existing Datadog estate | Scope and retention depend on the subscribed products | Shared telemetry can simplify correlation but complicate chargeback |
| Rollbar | Exception occurrence tracking and grouping | Validate framework setup and payload policy | Tag pipeline stage and team before ingestion |
| Healthchecks | Detecting late or missing scheduled work | It is not an exception store | Attribute checks by scheduled business process |
| Plain REST capture | Minimal dependency surface across HTTP and workers | Add separate heartbeats, alerts, and tracing where required | A stable adapter can attach cost center at every call site |

**Choose the smallest combination that closes the actual failure modes.** A team with mature Datadog instrumentation may gain little from another error store. A team needing browser replay or source-map processing should favor a product that explicitly supplies those capabilities. A small backend estate that values one HTTP contract and centralized grouping may reasonably choose plain REST capture plus Healthchecks, provided it accepts that alerting requires polling and distributed-trace queries live elsewhere.

Run a proof with representative failures before committing: one controller exception, one cron exception, one redelivered queue job, and one deliberately absent cron run. Confirm grouping, resolution, redaction, retry behavior, and cost-center attribution. Four tests reveal more than a long feature matrix.

## What should you deliberately stop keeping?

Retain enough to reproduce and assign a failure, not every byte that happened to be nearby. Drop duplicate payload copies, access tokens, full financial records, and high-cardinality context that has no debugging decision attached to it. Sample or aggregate repeated known failures only after verifying that the grouping key cannot merge distinct customer-impacting conditions.

The trade-off is real. Smaller events and shorter diagnostic retention reduce storage exposure and make attribution clearer, but an investigation may later lack the original input that triggered a rare parser failure. Preserve that input in the governed system of record under its own access and deletion policy, then store a correlation identifier with the error. This costs an extra lookup during an incident. It is preferable to turning an exception tracker into an accidental archive.

I would accept that lookup. I would not accept an error store becoming a second, weakly governed ledger merely because retaining whole inputs made the first incident easier to inspect.

Resolve groups; do not erase history merely to clean a dashboard. Resolution records the operational decision, while historical events preserve recurrence evidence. Reopening or regrouping behavior should be tested with each candidate because similar product vocabulary does not guarantee identical semantics.

## Further reading

- NestJS exception filters: https://docs.nestjs.com/exception-filters
- NestJS task scheduling: https://docs.nestjs.com/techniques/task-scheduling
- NestJS queues: https://docs.nestjs.com/techniques/queues
- Sentry NestJS documentation: https://docs.sentry.io/platforms/javascript/guides/nestjs/
- Datadog Error Tracking documentation: https://docs.datadoghq.com/error_tracking/
- Rollbar NestJS documentation: https://docs.rollbar.com/docs/nestjs
- Healthchecks documentation: https://healthchecks.io/docs/
- OpenTelemetry trace specification: https://opentelemetry.io/docs/specs/otel/trace/
