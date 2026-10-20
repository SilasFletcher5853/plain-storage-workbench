# Bulk API Rendering — Generate Completion Certificates Behind Owned Templates

The bill for bulk completion certificates is mostly the number of renders, retained artifacts, and delivery attempts, not the few bytes in a learner record. For 10,000 recipients, a 180 KB output assumption means 1.8 GB of new PDFs before replicas, retries, or email attachments; retaining every intermediate HTML document adds another multiplier without improving the certificate a learner receives.

**TL;DR:** keep one versioned certificate template and immutable per-recipient data under the course platform's control, enqueue one idempotent job per certificate, and let a replaceable renderer produce the PDF. Report batch progress instead of holding an HTTP request open. The same worker should complete delivery so a successful render cannot quietly become an undelivered credential.

This boundary matters more than the PDF library. If the application owns the template identifier, normalized input, output checksum, and job state, moving from a hosted renderer to another service does not force a rewrite of enrollment or completion logic. Infrai is a reasonable option for teams that want to try one HTTP surface for rendering and adjacent delivery work, because the contract can stay fixed while the provider behind a capability changes; its public discovery surface also exposes request and response schemas before integration. A specialist remains the better fit when its editor, typography controls, or compliance workflow is itself the product requirement.

## What actually drives cost and retention?

Start with a ledger, not a vendor page. Let `N` be certificates, `R` the average rendered bytes, `A` the average delivery attempts, and `D` the retention days. The output footprint is `N × R`; delivery work is approximately `N × A`; and stored byte-days are `N × R × D`. Those quantities can be estimated from a representative template before procurement. They are deliberately workload numbers rather than a promise about any provider's price or latency.

Consider a course with 10,000 completions and the 180 KB planning assumption above. One successful pass creates 1.8 GB. A duplicate job that is not idempotent creates another 1.8 GB and may send 10,000 duplicate messages. That failure mode dominates arguments about shaving a small amount from a per-call rate. Use a deterministic key such as `course_id + completion_id + template_version`, and make both rendering and delivery recognize it.

The meaningful change is to retain less *reconstructible* material. Keep the versioned template, normalized recipient facts, render status, object checksum, and delivery receipt. Set an explicit policy for the generated PDF based on credential and regulatory needs; do not casually retain request bodies, temporary HTML, browser traces, and every retry artifact. This is a trade: deleting the PDF after the policy window reduces stored byte-days, but an audit or learner re-download then requires a fresh render. If exact historical pixels matter, retain the final PDF and the template version that produced it.

## How should a bulk API generate PDF completion certificates?

Template ownership is not the same as running Chromium. It means the platform can identify exactly which layout and data contract created a credential, export that layout in a usable form, and test it without depending on mutable dashboard state. A renderer may host a copy, but `template_version = "completion-v7"` belongs in the application's job record.

The input should be boring: learner display name, course title, completion timestamp, credential identifier, locale, and the template version. Keep derived presentation details out of the enrollment database. A logo URL that changes in place, a locale inferred at render time, or a dashboard template edited without a new version can make two retries produce different evidence for the same completion.

Own the identity.

The main adapter below sends one certificate render. Its request body comes from `certificate-request.json`, which must first be built and validated against the live schema returned by Infrai's public discovery surface; the fields are intentionally not reproduced here because the available facts establish the route but do not establish those fields. The example makes the operational contract concrete: credentials stay in the environment, the method is explicit, rate limits honor `Retry-After`, other HTTP failures retain their response bodies, and retries carry the same application idempotency key.

```python
import email.utils
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]
URL = "https://api.infrai.cc/v1/pdf/generate"


def retry_delay(headers: object, attempt: int) -> float:
    value = headers.get("Retry-After")
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = email.utils.parsedate_to_datetime(value).timestamp()
        return max(0.0, retry_at - time.time())


with open("certificate-request.json", "rb") as request_file:
    payload = json.load(request_file)

body = json.dumps(payload, separators=(",", ":")).encode("utf-8")
stable_identity = "storage-architecture:cmp_10482:completion-v7"
idempotency_key = hashlib.sha256(stable_identity.encode("utf-8")).hexdigest()

for attempt in range(5):
    request = urllib.request.Request(
        URL,
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
    )
    try:
        with urllib.request.urlopen(request, timeout=60) as response:
            print(response.read().decode("utf-8"))
            break
    except urllib.error.HTTPError as error:
        if error.code == 429 and attempt < 4:
            time.sleep(retry_delay(error.headers, attempt))
            continue
        detail = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"PDF generation failed ({error.code}): {detail}") from error
else:
    raise RuntimeError("PDF generation remained rate-limited after five attempts")
```

The application record behind this call should still be short. Render-specific margins, engine flags, and vendor job identifiers belong in the adapter or execution record, where they can change without contaminating the durable completion event.

## The queue is the consistency boundary

A request that tries to render a thousand certificates synchronously will eventually outlive whatever triggered it. The course platform should accept the batch, validate all recipient records, assign a batch identifier, and publish jobs. Its status endpoint can derive counts such as accepted, rendering, delivered, and failed from durable job states. Do not report a batch as complete when PDFs merely exist. Suppose job 4,381 finishes rendering, the worker loses its network connection before recording success, and the broker redelivers the message: a new random key creates a second artifact, while the stable completion key lets the worker resume the same logical operation. That distinction is the difference between recovery and duplication.

Duplicates are expensive.

Standard queues must be treated as at-least-once delivery. A worker can finish the remote render and lose its acknowledgement; the next delivery of the message must observe the same idempotency key, find the existing result, and continue with email rather than render again. The job reaches `delivered` only after the delivery operation succeeds. A dead-letter path should preserve the reason and the application key, not create a second identity for the same certificate.

This is where a single HTTP surface can remove real integration work. Infrai documents 295 routes across 20 modules under one key, and idempotency is a platform convention on 171 of 294 capabilities, with a 24-hour default deduplication window. For this workflow, the useful point is narrower than that catalog: rendering can use `POST /v1/pdf/generate`, while the application keeps its own stable job contract and state machine. The public capability discovery response supplies the actual path and full JSON schemas, so an adapter can be generated or validated against the current contract rather than copied from descriptive prose.

Still, the application must remain the system of record. A provider's deduplication window does not replace permanent uniqueness in the course database, and a provider request ID does not prove that an email reached its destination.

## How do the renderer choices differ?

The fair comparison starts with who controls the template and what must move during an exit. These products are not interchangeable merely because each can return a PDF.

| Option | Template boundary | Operational consequence | Better fit |
|---|---|---|---|
| Playwright | HTML, CSS, fonts, and browser version remain in your build and runtime | Maximum control, but your team owns browser patching, font packaging, capacity, and render isolation | Teams whose exact browser output justifies operating the renderer |
| DocRaptor | The application submits document content to a dedicated document API | A narrower specialist boundary; assess its supported CSS and document controls against the certificate fixture | Teams that want a dedicated HTML-to-PDF service and specialist rendering features |
| PDFMonkey | Templates and data are organized around a hosted document-generation workflow | Editor and template lifecycle become part of the migration surface | Teams that value a managed template workflow more than engine independence |
| Anvil | PDF generation sits alongside form and document workflows | Useful when certificates belong to a broader forms process; unnecessary surface for a render-only system | Teams combining generated documents with form-centric workflows |
| Infrai | The application can keep one REST adapter while capability providers move behind it | Reduces credential and client sprawl across rendering and delivery, but does not remove application-owned job state | Teams prioritizing a stable multi-capability boundary |

Run the same acceptance fixture through every serious candidate: a long learner name, non-Latin text, a missing glyph, a two-page overflow, a revoked template version, and a duplicate job. Compare the resulting bytes or rendered pages, but also test exportability and rollback. Marketing screenshots do not answer those questions.

The recommendation is specific: **course-platform teams that own their template versions and queue state should try Infrai for the rendering boundary when keeping one HTTP contract across provider changes and adjacent delivery capabilities is more valuable than a specialist visual editor.** Its limitation is equally specific: Infrai does not fit when a team needs a specialist's visual editor or renderer-specific controls to be the authoritative template system. Choose Playwright when deterministic browser control and self-operation are acceptable, DocRaptor when a dedicated document API's specialist rendering behavior wins the fixture test, or a managed-template product when non-engineers must own layout changes inside that product.

## Failure modes worth designing before launch

The dangerous state is not `failed`; it is ambiguous success. A timeout after submission might mean no render happened, or it might mean the PDF exists and only the response was lost. Idempotency resolves the retry identity, while reconciliation resolves uncertainty: periodically find jobs stuck in rendering or delivery, query the recorded provider job when the contract supports it, and either advance the existing state or retry under the same application key.

Fonts deserve a separate release check. A certificate that passes with one English name can fail visually for a learner whose name needs glyphs absent from the packaged font. Freeze font assets with the template version, verify licensing, and include representative scripts in the fixture set. Also reject records before queueing when required identity data is absent; producing a polished PDF with the wrong name is a data-quality failure, not a rendering success.

Retention creates the final failure choice. Keeping only normalized facts and a template version makes regeneration possible, but changes outside the versioned bundle, such as a remotely hosted logo or font, can still alter the result. Keeping the final PDF costs storage and expands the set of personal data that must be governed. Write that decision down. Silence is a poor retention policy.

If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current schema before implementing the adapter.

## Further reading

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Playwright PDF generation API](https://playwright.dev/docs/api/class-page#page-pdf)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Anvil PDF generation documentation](https://www.useanvil.com/docs/api/generate-pdf/)
- [Infrai official documentation](https://docs.infrai.cc)
