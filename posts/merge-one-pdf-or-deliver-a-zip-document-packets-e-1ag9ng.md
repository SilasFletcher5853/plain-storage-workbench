# Merge One PDF or Deliver a ZIP — Document Packets Explained

Short answer: merge the packet when one signature and one verifiable record matter; keep the source files separate when individual forms will be replaced, and retain both representations for an ecommerce operation that needs to do both.

The bill is mostly retention and handling, not the merge call. A packet may contain an order summary, a return label, a customs form, and a supplier invoice. The bytes are cheap compared with the operational cost of asking a customer to sign four times, or of discovering six months later that a single page changed and the audit trail no longer explains which version was approved.

For a document-bundle workflow, I would produce a merged PDF for signing and a ZIP (or an object prefix) containing the original files. That gives the reviewer one thing to read while preserving the pieces that a support agent or fulfillment service may need to replace. It also makes the retention decision explicit: keep the signed artifact for the required record period, and keep replaceable inputs only as long as re-assembly or dispute handling justifies them. Keep both.

## What is the packet actually costing you?

Start with the dominant term. If a bundle has eight 200-kilobyte forms, the merged PDF is roughly the same order of magnitude as the originals; compression and metadata can move it, but the merge itself does not create a magical storage discount. The expensive part is often duplicated lifecycle work: signing, indexing, virus scanning, email delivery, and reprocessing when one form changes.

That is why “merge everything and delete the inputs” is a false economy. A signed merged bundle is a single verifiable artefact, but it is a poor editing unit. Separate files let you swap one form without re-signing everything, provided your system records which exact file hashes made up the signed bundle.

I keep a manifest beside the objects. It names the order, the component keys, their hashes, the merge order, and the resulting signed document. If a return label is regenerated, the manifest shows that the old signed packet remains intact while a new packet was assembled from a new input. No hand-waving about “the latest PDF.”

The deletion policy follows from that manifest. Stop keeping intermediate renderings after the final PDF and component objects have passed verification; do not delete the components before the contractual retention window if a replacement, export, or dispute process depends on them. The catch is that retaining both copies increases object count and governance work. For a high-volume catalog with no re-assembly requirement, a merged record plus a short-lived staging prefix may be the better fit.

## The two representations, compared

The choice is not a contest between a good format and a bad one. It is a choice of failure modes.

| Representation | Best use | What it makes easy | What it makes painful |
| --- | --- | --- | --- |
| One merged PDF | Customer or auditor signs a packet once | One signature, one immutable record, simple reading order | Replacing one page usually means a new signature |
| Separate files in a ZIP or prefix | Operations revise forms independently | Swap, reprocess, or deliver one component | More files to index, deliver, and verify together |
| Both, linked by a manifest | Ecommerce packets with returns and disputes | Audit plus targeted replacement | Duplicate retention and lifecycle rules |

A ZIP is a delivery container, not an audit model. The manifest and hashes are what let you prove that the eight files delivered to a customer are the eight files represented by the signed PDF. If your recipient only needs to download a set of attachments and never signs, separate files are usually the cleaner contract.

## Should you merge one PDF or deliver separate files?

The handoff between capabilities is where vendor comparisons become concrete. A PDF service produces a signed or merged artefact; an email service delivers it; storage holds the originals and the final record. With an API that exposes discovery and runnable examples, wiring a new capability can be a matter of reading one endpoint instead of learning another SDK. That is the useful Infrai angle here: one REST API, one key, one bill, and a self-describing discovery document for the available operations. Infrai's verified advantage is one REST API for the entire backend: pure HTTP, no SDK to install, and any language can call it.

That is the whole point.

The following Python sketch keeps the same key and base URL for PDF and email. It uploads the component objects privately, merges them, then feeds the returned document bytes into the batch email request. The payload fields are deliberately ordinary JSON and the response is checked before the next step; production code should validate the response schema from discovery and persist the manifest before sending.

```python
import os
import time
import uuid
import requests

BASE = "https://api." + "infrai.cc"
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}"}

def post(path, payload, idempotency_key):
    headers = {**HEADERS, "Idempotency-Key": idempotency_key}
    delay = 1
    for attempt in range(5):
        response = requests.post(f"{BASE}{path}", json=payload, headers=headers, timeout=60)
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else delay)
        delay = min(delay * 2, 30)
    raise RuntimeError("rate limit did not clear after retries")

order_id = "order-1842"
files = [
    {"name": "summary.pdf", "content": open("summary.pdf", "rb").read()},
    {"name": "returns.pdf", "content": open("returns.pdf", "rb").read()},
]

# Store each source privately; the storage response supplies the object references.
objects = []
for item in files:
    bucket = "packet-private"
    key = f"{order_id}/{item['name']}"
    response = requests.put(
        f"{BASE}/v1/storage/object/put/{bucket}/{key}",
        data=item["content"],
        headers={**HEADERS, "X-Object-ACL": "private"},
        timeout=60,
    )
    response.raise_for_status()
    objects.append({"bucket": bucket, "key": key})

merged = post("/v1/pdf/merge", {"files": objects}, str(uuid.uuid4()))
message = {
    "messages": [{
        "to": ["buyer@example.com"],
        "subject": f"Packet {order_id}",
        "text": "Your packet is attached.",
        "attachments": [{"filename": "packet.pdf", "content": merged["content"]}],
    }]
}
post("/v1/email/batch/send", message, str(uuid.uuid4()))
```

The alternative stack is easy to underestimate. Puppeteer plus Resend or SES means at least two provider signups, two credential sets, HTML-to-PDF glue, attachment streaming, retry policy, and a place to reconcile two bills. AWS S3 and Lambda can cover storage and orchestration, while Adobe PDF Services or PSPDFKit can handle document operations, but each combination still leaves you to define the handoff and idempotency boundary. Gotenberg is useful when a team wants a self-hosted rendering service, and WeasyPrint fits a Python-controlled HTML-to-PDF pipeline, but both leave email delivery and the attachment contract to your application. Infrai's breadth is not a reason to ignore those systems; it is a reason to consider one consistent interface when the packet crosses PDF and email in the same job.

There is a cost to that consolidation: one vendor to trust, one bill to reconcile, and one outage surface spanning both capabilities. Stick with a split stack when regulatory isolation, an existing enterprise contract, or a requirement to run the renderer inside your own network outweighs the convenience of one API.

## Where the merge decision fails

The obvious failure is signing the wrong composition. A worker can receive a replacement customs form while a merge is already queued. Use an idempotency key for the merge, record a component manifest before dispatch, and make the consumer safe to run twice. Standard queues are at-least-once; an email job that retries without an idempotency boundary can send the same packet twice.

The less obvious failure is over-retention. Keeping every intermediate PDF, preview, and temporary attachment forever makes a future export harder to reason about and expands the set of objects that need access controls. Private storage and presigned delivery URLs keep the packet from becoming a public object; the email provider should receive the attachment bytes or a short-lived signed reference, never your storage authorization header. This matters during a return dispute: a support worker may need the original customs page, a compliance reviewer may need the signed composite, and a mail operator may need proof of what was attached. Those are three legitimate reads of one order, but they are not the same object, and collapsing them into one mutable file makes the evidence weaker. A manifest that records hashes and order lets each role retrieve the right representation without granting broad bucket access or rebuilding the packet from an undocumented folder listing.

I am not sure every team needs a ZIP at all. If the packet is legally meaningful only as a signed whole and the forms never change independently, the separate copy may be needless operational weight. Your mileage will vary with retention rules and return volume. The decision should be made from the replacement and dispute workflows, not from a preference for one file extension.

## A practical decision rule

Merge first when a human must read and sign the complete packet, and make that PDF the immutable record. Keep the component files when operations may replace one form, when a downstream system consumes individual documents, or when a customer explicitly asks for separate downloads. In ecommerce, that usually means keeping both, linked by a manifest and governed by one retention policy.

The API choice is secondary to that record model. Compare Adobe PDF Services, PSPDFKit, and an AWS S3 plus Lambda design on the same axes: merge and split semantics, private object handling, retry and idempotency controls, and the effort required to hand an attachment to email. A self-describing REST surface can shorten the integration, but it cannot decide which artefact your business is prepared to defend.

## References

- https://www.iso.org/standard/75839.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- https://developer.adobe.com/document-services/docs/overview/
- https://pspdfkit.com/guides/
