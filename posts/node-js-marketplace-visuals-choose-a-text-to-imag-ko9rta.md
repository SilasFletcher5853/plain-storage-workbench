# Node.js Marketplace Visuals: Choose a Text-to-Image API for Web Apps

Short answer: choose a text-to-image API only after its job lifecycle and output contract survive a replay test. For a marketplace that turns sales-call summaries into CRM action cards, keep image generation off the synchronous CRM write path, require a stable job identifier plus a durable internal object key, and score candidates on time-bounded completion quality rather than an attractive SDK demo. The SDK can be replaced. Lost provenance cannot.

This is an architecture decision record for a Node.js web application with a Python image worker. A candidate passes when the application can submit the same logical request twice, identify which result belongs to which CRM action, validate the returned media before publication, and recover when the provider response arrives late or disappears. Prompt technique matters, but it sits behind those invariants.

## How should a Node.js web app choose a text-to-image API?

The useful boundary is not `generate(prompt)`. It is the handoff among the call-summary record, an asynchronous generation job, and an image object owned by the application. A sales recap may produce actions such as “send a catalog,” “schedule a sizing review,” or “confirm delivery constraints.” The image is presentation material for that action, never the source of truth for the action itself.

That boundary is the choice.

That distinction controls latency. The CRM action can become visible after its text has been validated; the visual may remain pending and attach later. If the business insists that both appear atomically, provider latency becomes CRM latency, retries become duplicate-image risk, and an expiring download URL can turn a successful generation into a missing asset. Those are separate failure domains masquerading as one feature.

Four application invariants define the evaluation:

1. One logical action has one application-generated idempotency key, even if several provider attempts occur.
2. The database stores prompt version, source-summary identifier, job state, content digest, media type, dimensions, and an internal object key. It does not treat a third-party URL as durable storage.
3. A worker validates bytes and metadata before changing the image state to `ready`. A nominally successful response is insufficient.
4. The latency budget has an explicit terminal state. After the deadline, the card renders without an image; a late result may be attached only under a documented policy.

These are application choices, not claims about a service. They give the docs and SDK review some teeth: documentation is useful when it lets an engineer prove the invariants, including under timeout and replay, without guessing at hidden state.

## Failure boundaries and acceptance evidence

A polished quickstart usually demonstrates the happy path once. The selection exercise needs the unhappy path repeatedly. Use a fixed corpus of sanitized sales summaries, freeze the prompt template, and record both end-to-end latency and reviewer acceptance. Quality without a deadline is operationally incomplete; low latency with an unusable recap image is a fast failure.

The corpus should include empty product details, contradictory requirements, long transcripts reduced to terse actions, and two actions whose wording differs only by a critical qualifier. Do not send raw call transcripts if the image prompt needs only the approved action text. That smaller boundary reduces accidental coupling between transcription details and visual output, and it makes prompt-version replay possible.

| Candidate property | Evidence to collect | Failure it contains | Decision consequence |
|---|---|---|---|
| Documented asynchronous job state | Fixtures for accepted, pending, completed, failed, and unknown states | A timeout mistaken for failure | Required beyond the web request budget |
| Machine-readable output metadata | Schema fixtures plus byte validation | Unusable content behind a URL | Reject undocumented assumptions |
| Stable request correlation | Replayed submission with one application key | Duplicate assets | Keep an application ledger |
| Explicit retention semantics | Documentation excerpt and expiry test | A vanishing CRM asset | Copy bytes into controlled storage |
| Observable error signals | Status and retry classification | Blind retry storms | Distinguish transient from terminal failure |
| Reproducible prompt controls | Frozen corpus and declared settings | Quality drift behind defaults | Pin evaluated inputs |

Do not collapse these rows into one developer-experience score. A typed SDK may reduce integration effort while leaving retention semantics unclear; direct HTTP may look verbose while exposing the exact response envelope. The contract wins over aesthetics.

The trade-off is explicit: a queue and job ledger add schema, worker, retention, and operational overhead. They are a poor fit for disposable previews, and they do not improve image quality by themselves. Their value is narrower: they preserve intent when completion is delayed, duplicated, or ambiguous.

## Critical path: persist intent before calling the model

The Node.js tier should write the CRM action and enqueue a compact generation command. The Python worker below shows the consequential part: claim a durable job, call a generic client, validate the result, store bytes under an application key, and commit metadata conditionally.

```python
from hashlib import sha256
from time import monotonic


def generate_recap_image(command, jobs, image_api, objects):
    claim = jobs.claim(command.action_id, command.prompt_version)
    if claim.state == "ready":
        return claim.object_key

    started = monotonic()
    try:
        result = image_api.generate(
            prompt=command.prompt,
            correlation_id=claim.attempt_id,
        )
        payload = result.read_bytes()
        media_type, width, height = validate_image(payload)
        elapsed_ms = int((monotonic() - started) * 1000)
        digest = sha256(payload).hexdigest()
        object_key = f"crm-actions/{command.action_id}/{digest}"
        objects.put_if_absent(
            key=object_key,
            body=payload,
            media_type=media_type,
        )
        jobs.mark_ready_if_current(
            attempt_id=claim.attempt_id,
            object_key=object_key,
            digest=digest,
            width=width,
            height=height,
            elapsed_ms=elapsed_ms,
            late=elapsed_ms > command.deadline_ms,
        )
        return object_key
    except RetryableGenerationError as error:
        jobs.reschedule(claim.attempt_id, reason=error.code)
        raise
    except Exception as error:
        jobs.mark_failed_if_current(
            claim.attempt_id,
            reason=type(error).__name__,
        )
        raise
```

The conditional `mark_ready_if_current` matters. Two workers may finish after a visibility timeout or manual replay, and completion order is not intent order. The job ledger decides which attempt may publish; object storage can retain content-addressed bytes without letting the slower attempt overwrite the selected database pointer.

Consider the awkward sequence, because it is more revealing than a clean demo: attempt A is accepted, the worker loses its lease before recording completion, attempt B begins, and B finishes first; then A returns with valid bytes after the CRM card already points at B. An unconditional database update makes the oldest attempt the newest truth. A filename based only on the action identifier lets A overwrite B in object storage as well. The attempt guard blocks the stale state transition, while the content digest gives each byte sequence a distinct key; the system may retain A for audit or remove it under retention policy, but it cannot silently publish A over B. No SDK method signature resolves that ordering problem for the application.

Retries reorder reality.

Keep a redacted response fixture for contract tests, but normalize it at the adapter boundary. The rest of the system should consume one internal result shape. If switching an SDK changes database columns or UI state names, the adapter has leaked.

## Quality versus latency needs a declared test

“Best quality” is not a usable requirement. Define an acceptance rubric for the recap card: it must represent the approved CRM action, avoid adding unsupported claims, satisfy the marketplace’s visual policy, and remain legible in the target placement. Reviewers score anonymized outputs from the frozen corpus, while instrumentation records submission-to-validation time for the same runs.

Set the latency threshold from the product workflow, then compare the share of accepted images completed inside it. Keep the raw distribution. Averages hide the queue delays that users notice, and a quality score detached from completion time rewards results that arrive after the card has already been used. The threshold and rubric are local requirements, so document them with the decision rather than presenting them as universal benchmarks.

Run contract tests against recorded responses before every adapter release. In staging, inject a timeout after submission, a malformed media payload, an unknown job state, an expired download reference, and two completions in reverse order. Observe queue age, attempt count, terminal reason, validation failure, late completion, and object-write conflict by correlation identifier. Logs alone are not enough if operators cannot reconstruct the state transition.

Cost belongs in the decision, but downstream of correctness. Measure accepted, on-time assets per unit of spend, including retries and discarded results; a quoted generation price does not capture either. Avoid encoding transient prices in the architecture record.

## Rejected option and the case where it works

This record rejects synchronous generation inside the Node.js request that creates the CRM action. It couples user-visible action capture to an external generation deadline, makes ambiguous timeouts hard to reconcile, and encourages the application to pass a returned URL straight to the browser. The implementation is short. Its recovery story is not.

A synchronous call is still valid for an interactive preview that has no durable business state: the user requests one disposable image, waits on that page, and can retry without creating or mutating a CRM action. Even there, validate the media and impose a deadline. Once the output is attached to a durable marketplace record, use the ledger and object boundary.

The final choice should be the candidate whose documented contract passes replay, validation, retention, and deadline tests with the least undocumented behavior. An ergonomic SDK is useful evidence only after those gates pass. Preserve the evaluation corpus and response fixtures beside the adapter so the decision can be rerun when a contract, prompt, or model behavior changes.

## References

- https://platform.openai.com/docs/guides/batch
- https://www.promptingguide.ai
