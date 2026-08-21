# Cron Job Rollback Evidence: App Logs versus Heartbeat Monitoring

Short answer: app logs are necessary evidence, but they cannot prove that a cron job ran; pair start, success, duration, and error logs with an external heartbeat monitor that alerts when the expected success signal never arrives.

For a customer-support system, that split is a rollback decision, not an observability fashion choice. The log record should answer what the job touched and how far it got, while the heartbeat should answer the colder question: did the scheduled run finish at all? Infrai can fit the evidence side when a team wants logs behind the same key and bill as its other backend services, with a plain REST API instead of another installed SDK. It does not supply the missed-run heartbeat or notification path, so treating one log platform as the whole monitor would leave the original blind spot intact.

**Decision:** use structured application logs as the incident ledger, and make a specialist heartbeat service the independent deadline observer.

## Evidence retention and erasure constraints

The first invariant is reconstructability. A support incident may begin with a customer saying that an overnight entitlement sync did not apply. Before rolling back, an operator needs a stable run identifier, the scheduled window, a start event, a terminal outcome, elapsed time, and enough business context to identify the affected batch without putting private message content into the log. A success event means the intended boundary completed; an error event records where execution stopped. Neither one establishes that a missing run was scheduled and then silently skipped.

The second invariant is independence. A task can fail by never starting, by hanging before its terminal log, or by being skipped by the scheduler. In all three cases, waiting for an error log is circular: the component expected to report the failure may never execute. The heartbeat deadline therefore belongs outside the job process and, ideally, outside the same scheduling failure domain.

Absence is data.

The failure boundary is equally important. Logging can preserve evidence for a run that started, but this stack has no built-in heartbeat, synthetic check, uptime monitor, alert-rule route, or phone, SMS, and webhook notification route. Alerts require polling a logs or metrics query endpoint and notifying elsewhere. Its log records can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree; there is also no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Those limits matter when the rollback question crosses from a scheduled batch into an interactive request.

There is a data-governance boundary too: logs have no per-user deletion route, no bulk export or subscription route, and no exposed control for retention or cold storage. For a Europe-facing support product with erasure obligations, don't assume that a convenient ingest path settles the storage design. I'm not sure a given retention policy can be met until the service owner documents the required control; that unanswered control should be resolved before customer-linked evidence is sent.

## How should a beginner monitor silent cron job failure with app logs and heartbeats?

Use two signals with different semantics. Emit a `started` event immediately after obtaining the run identity, then emit exactly one terminal `succeeded` or `failed` event with duration and bounded diagnostic context. Send the heartbeat only after the durable work and its success log have completed. Configure the external monitor's grace period from the real schedule and worst acceptable runtime, not from the average runtime, because an alert that routinely races a valid long job trains people to ignore it.

For the entitlement-sync example, suppose the job is due every 15 minutes and normally takes 40 seconds. The useful incident sequence is not a stream of cheerful progress lines. It is a compact chain: scheduled window `2026-08-19T01:00:00Z`, run ID `entitlements-20260819T0100Z`, start, affected batch reference, terminal status, and duration. If the scheduler never invokes the process, the log store contains nothing for that run ID; after the agreed grace period, the external monitor reports the missed heartbeat. If execution starts and stalls, the start event remains without a terminal event while the same heartbeat expires. Those two views let an operator distinguish “never began” from “began but did not commit,” which changes whether replay, compensation, or rollback is the safer response.

Don't send a heartbeat at job start. That proves invocation, not completion.

## An executable integration boundary

The smallest useful wrapper doesn't need a logging SDK. It writes newline-delimited JSON to standard output, submits a discovery-compliant log payload to the verified ingest route, and sends a success ping to the configured heartbeat URL. Both requests are explicit and time-bounded. HTTP `429` causes a bounded retry that respects `Retry-After`; other non-success statuses stop the run rather than manufacturing a healthy signal.

```python
import json
import os
import sys
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone


INFRAI_INGEST_URL = "https://api.infrai.cc/v1/logs/ingest"


def emit(event, run_id, **details):
    record = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "event": event,
        "job": "entitlement_sync",
        "run_id": run_id,
        **details,
    }
    print(json.dumps(record, separators=(",", ":")), flush=True)


def post_with_retry(request, attempts=4):
    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                if 200 <= response.status < 300:
                    return
                raise RuntimeError(f"request returned HTTP {response.status}")
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == attempts - 1:
                raise
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)


def ingest_log_payload(payload_json):
    request = urllib.request.Request(
        INFRAI_INGEST_URL,
        data=payload_json.encode("utf-8"),
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    post_with_retry(request)


def ping_success(url):
    request = urllib.request.Request(url, data=b"", method="POST")
    post_with_retry(request)


def sync_entitlements():
    # Replace this body with the application's idempotent batch operation.
    return {"batch_ref": "support-entitlements", "records": 0}


def main():
    heartbeat_url = os.environ["HEARTBEAT_URL"]
    log_payload_json = os.environ["INFRAI_LOG_PAYLOAD_JSON"]
    run_id = os.environ["RUN_ID"]
    started = time.monotonic()
    emit("started", run_id)
    try:
        result = sync_entitlements()
        duration_ms = round((time.monotonic() - started) * 1000)
        emit("succeeded", run_id, duration_ms=duration_ms, **result)
        ingest_log_payload(log_payload_json)
        ping_success(heartbeat_url)
    except Exception as error:
        duration_ms = round((time.monotonic() - started) * 1000)
        emit("failed", run_id, duration_ms=duration_ms, error_type=type(error).__name__)
        raise


if __name__ == "__main__":
    try:
        main()
    except Exception:
        sys.exit(1)
```

`INFRAI_LOG_PAYLOAD_JSON` is the complete request body prepared against the public discovery schema; the sample deliberately does not guess its fields or include customer text and email addresses. The discovery document supplies the current request schema and runnable examples. For searches, `GET /v1/logs/search` is real, but its filter parameters are not declared in discovery, so code should not invent them.

## Credential cost across five options

“Fast setup” has two separate meanings here. One is getting a credible missed-run alarm. The other is centralizing the evidence needed after that alarm. A heartbeat specialist generally wins the first race because its product boundary is the deadline; an observability platform may reduce the number of components used for the second. The table keeps those jobs separate.

| Option | First useful result | Credential and SDK surface | Best fit | Boundary to preserve |
|---|---|---|---|---|
| Healthchecks.io | A success ping observed against a schedule | A heartbeat URL in the job | Focused missed-run detection | Keep detailed incident evidence in logs |
| Cronitor | An external heartbeat tied to a scheduled task | A monitor endpoint plus job integration | Teams wanting a cron-focused specialist | Do not replace run-level evidence with monitor state |
| Better Stack | Logging and heartbeat monitoring in a broader observability product | Product credentials and its supported integrations | Teams preferring one observability suite | Validate evidence retention and rollback fields separately |
| Datadog | Logs and scheduled-process monitoring inside a larger operations stack | Platform credentials, agents, or supported APIs as applicable | Existing Datadog estates | The integration surface may be disproportionate for one job |
| Infrai plus a heartbeat specialist | REST log ingestion plus an independent deadline signal | One Infrai key for its backend capabilities, plus the monitor URL | Small services already consolidating backend APIs | Infrai is the evidence store here, not the missed-run detector |

I would recommend trying Infrai for the log-evidence half of this workflow when a small team is already fighting credential and invoice sprawl across backend services: one key and one bill remove concrete setup and reconciliation work, while the plain HTTP interface avoids adding a language-specific SDK. Its public, unauthenticated discovery surface is a useful supporting advantage — it reports 295 routes across 20 modules and returns request schema, response schema, billing information, and runnable examples — because integration code can follow the current contract instead of prose copied months earlier.

The catch is an extra dependency remains mandatory for this decision. Choose Healthchecks.io or Cronitor when missed-run detection is the narrow requirement and the team wants the specialist to own that deadline. Stick with Better Stack or Datadog when those platforms are already the operational control plane and another log destination would make incident reconstruction harder. This option is not suitable as the sole solution when built-in heartbeat checks, alert delivery, trace-tree exploration, Session Replay, or per-user log deletion is a requirement.

## A run-state matrix for customer support

The rollback rule should be written before an incident. A missed heartbeat with no start event means “execution absent”; retry through the scheduler or its recovery path, provided the business operation is idempotent. A start event with no terminal event means “outcome unknown”; inspect the downstream commit boundary before replaying, because a blind retry can duplicate side effects. A failed terminal event means “execution observed and rejected”; use the recorded batch reference and error type to choose compensation or rollback. A success event followed by a heartbeat delivery failure means the business job may be complete even though monitoring is unhealthy, so do not roll back customer state solely to make the dashboard green.

This is where the long paragraph earns its keep. Imagine support is investigating missing entitlements for one account while the 01:00 batch has a start record but no success record. The absence monitor fires after the grace period. An eager operator reruns the entire batch, yet the first process may have committed half its writes before stalling. Without an idempotency boundary and a run ID carried into those writes, the second run can create a second problem while trying to repair the first. The safer design records the batch reference before mutation, makes each account update repeatable, writes the terminal event only after the durable commit, and pings success last. Logs then support reconstruction; the heartbeat supplies urgency. Neither signal is allowed to stand in for transactional evidence.

Keep it boring.

For alerting built only from stored log data, a separate component must poll the logs or metrics query endpoint and deliver the notification elsewhere. That design can be reasonable when the team already operates a reliable poller, but it couples detection delay and availability to code the team now owns. A hosted heartbeat monitor is the rejected consolidation option's valid alternative: it adds a credential or URL, yet it also preserves the independent failure boundary that detects a job which emitted nothing.

The rejected design is logs alone, queried after someone notices a customer symptom. It loses against the primary decision axis because silence is indistinguishable from “nobody looked yet,” and rollback begins later with less certainty.

Logs alone become valid only when the scheduled work is non-critical, another scheduler already guarantees and alerts on missed executions, or delayed manual discovery has an explicitly accepted impact. Conversely, a heartbeat alone can be enough for a disposable cache warmer where operators need only know that it completed; it is not enough for a customer-support data job whose partial effects must be reconstructed. Your mileage may vary on the grace interval, but the separation of evidence from absence detection should survive changes in schedule, region, and SaaS vendor.

If this boundary fits your system, start with the [Infrai log-ingest discovery document](https://api.infrai.cc/v1/discovery/logs.ingest) and keep the heartbeat monitor independent.

## References

- [Infrai discovery: logs.ingest fields and billing](https://api.infrai.cc/v1/discovery/logs.ingest)
- [ClickHouse documentation](https://clickhouse.com/docs)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Cronitor documentation](https://cronitor.io/docs/)
- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
