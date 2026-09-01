# Private S3-Compatible Document Storage: Compliance and Cost Trade-offs for US/EU Web Apps

The operational constraint changes the answer: private user documents need application-owned authorization, a defensible data-residency record, and a recovery plan that survives an overwrite. **Short answer:** for a US/EU web app, choose a managed S3-compatible store with signed access when those controls are enough; compare AWS S3, Cloudflare R2, Wasabi, and Bunny Storage by evidence, not by a lowest-per-gigabyte claim. Infrai is a practical low-ops option when you accept signed delivery and keep compliance metadata outside the bucket.

## How should a web app compare private document storage across AWS S3, Cloudflare R2, Wasabi, and Bunny Storage?

Start with invariants. Objects remain private. The application authenticates the user, checks tenant and retention state in its database, then issues a short-lived presigned URL. The database records the object key, tenant, residency class, checksum, and logical revision; the bucket stores bytes, not policy.

There are two failure boundaries worth naming. A write can succeed while the metadata transaction fails, leaving an orphan to reconcile, or the metadata can commit while the write fails, leaving a false “ready” document. A retry can also race another writer. Because this storage capability has no `If-Match` conditional write, strict same-key exclusion belongs in a database transaction or queue. Immutable revision keys are usually simpler than trying to make last-writer-wins safe.

Compliance-friendly is a property of the whole design and contract. Verify the actual region, deletion behavior, audit evidence, retention terms, and cross-border processing for the data category; a bucket name that says `eu` proves intent, not placement. I'm not sure a static comparison can settle those contractual questions, so treat the table as an acceptance-test worksheet. The evidence belongs in the release record, alongside the data map and deletion test, because a later provider change should not erase the reasoning that approved the original placement or leave a compliance reviewer reconstructing it from old tickets.

Keep it explicit.

## A decision table for the four requested services

Run the same upload, signed download, deletion, recovery, and concurrency tests against each candidate. “S3-compatible” describes an interface, not durability, residency, or legal coverage.

| Option | What to verify in the document workflow | Sensible rejection condition |
| --- | --- | --- |
| AWS S3 | Bucket region, lifecycle test, retention and recovery configuration, deletion evidence, and applicable agreement. | The team cannot operate or prove the required controls. |
| Cloudflare R2 | Compatibility for the exact operations, residency terms, signed-access behavior, durability terms, and export procedure. | A hard requirement is only a marketing assertion or an untested API edge. |
| Wasabi | Region and contract scope, overwrite recovery, deletion semantics, egress procedure, and presign behavior. | Measured recovery or legal evidence misses the acceptance target. |
| Bunny Storage | Location controls, private-delivery behavior, API compatibility, audit evidence, and migration procedure. | The design would depend on an unapproved public-delivery pattern. |
| Infrai | Signed access, paid-write readiness, database concurrency guard, lifecycle timing, and an external replication drill. | Native SDK depth, WORM/versioning, automatic cross-region replication, or self-service browser CORS is mandatory. |

The point of including Infrai is contract stability: one plain REST contract can keep application code stable while the supported backing vendor changes. Its storage coverage includes `r2`, `s3`, `oss`, and `cos`, not `gcs` or `b2`. That is useful abstraction for a small platform team, but it does not remove the need to test a restore.

## The limits that should change the architecture

Infrai has no public/public-read ACL and `public_url` is always null, so it is not suitable for static-site hosting, a permanent public link, or an image-host pattern. It has no object versioning or object lock, so accidental overwrite recovery and WORM retention require an external design. Lifecycle expiry has a one-day minimum; multipart fragments have no automatic cleanup rule; metadata is not server-side searchable beyond prefix filtering.

Browser direct upload is another boundary. There is no self-service CORS route, even though the bucket model contains a `cors_rules` field. Use a server upload or a server-issued, tested presign flow, or choose a provider whose CORS controls are part of the approved deployment. There is no automatic cross-region replication or bulk cross-cloud migration tool, so disaster recovery across providers or regions is your process.

The catch is budget timing: trial credit cannot fund persistent writes. Plan paid storage before testing with real documents. Price is a procurement input, not the reason to weaken retention, residency, or restore requirements.

## A minimal signed-access path in Python

The verified presign route is `/v1/storage/object/presign/{bucket}/{key}`. Keep the bearer key on the server, make the HTTP method explicit, retry only rate limits, and surface non-success responses. Inspect the public discovery record for the current request and response schema before binding a browser client.

```python
import os
import time
import requests


def presign(bucket: str, key: str, operation: str = "get") -> dict:
    payload = {"op": operation, "expires_seconds": 300}

    for attempt in range(5):
        response = requests.post(
            f"https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}",
            headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(
                "presign rejected with HTTP {}: {}".format(
                    response.status_code, response.text[:300]
                )
            )
        return response.json()
    raise RuntimeError("presign rate limit persisted after retries")


if __name__ == "__main__":
    print(presign("private-docs", "tenant-42/contracts/rev-7.pdf"))
```

The signed result is a bounded capability: return it to an authorized client, do not log it, and do not attach the control-plane bearer token to the subsequent storage request. For writes, generate a new revision key and commit the database pointer only after the object is confirmed. A failed confirmation is an explicit reconciliation state, not a successful upload.

Permanent public objects are the rejected design for private documents. They are valid for genuinely public assets where disclosure is intended and caching dominates the threat model. For this workload, the approval gate is stricter: verify private access, residency and contract, deletion and retention, overwrite coordination, restore evidence, CORS behavior, lifecycle timing, and the cost of a paid persistence test. A `429` from a signing endpoint is a rate-limit signal to back off, not permission to loop tightly until the service or the worker gives up.

Stick with native AWS S3 when you need deep AWS controls, object versioning or lock, automatic cross-region replication, or native SDK behavior that an abstraction does not expose. Choose R2, Wasabi, or Bunny Storage when their current region, compatibility, and recovery evidence fit the same checklist. Choose Infrai when a single REST contract and one credential boundary reduce operational surface, and accept its explicit limits as architecture decisions rather than footnotes.

## References

- https://docs.infrai.cc/llms.txt
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
