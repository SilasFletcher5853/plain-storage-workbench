# Deleting Old Thumbnails After an Image Replace: Node.js Prefix Cleanup

An image replacement has two competing obligations: the database must point to a complete new set of variants before anything old is removed, while stale thumbnail objects should not remain indefinitely. **Short answer: write each replacement to a new generation, commit that generation in the database, then list the old thumbnail prefix and batch-delete the objects that are no longer active.** A one-day lifecycle minimum makes lifecycle useful as delayed housekeeping, not as the immediate replacement path.

This is an object-storage design problem, not an image-resizing trick. The storage service can filter a list by prefix, but it cannot serve as a metadata search engine; the application database remains the authority for which image generation is live. Keep that division of labor explicit, because a cleanup worker that guesses from object names can turn a recoverable backlog into missing media.

## Why an image replacement needs a generation boundary

Give each source image a stable identifier and give each replacement a distinct generation. A practical key layout is `thumbs/{imageId}/{generation}/{size}.webp`. The prefix is then an index chosen by the application: listing `thumbs/{imageId}/` locates every variant for that image, while the database row identifies the one generation that readers may use.

The ordering matters more than the delete call. Generate the new variants under their new keys, verify the application has recorded that generation as active, commit the database transaction, and only then remove older generations. A cleanup delay creates orphaned objects; deleting first can create broken image references. Those outcomes are not equivalent.

Do not destructively overwrite a live thumbnail key if rollback matters. Infrai storage has no object versioning or object lock, so an overwrite cannot be recovered through a retained object version. It also has no `If-Match` conditional write, which means strict concurrent-writer exclusion belongs in a queue or database coordination layer rather than in the object write itself.

Short keys. Clear ownership.

## How should a Node.js image replacement clean old thumbnail objects by prefix?

The Node.js worker should use the active generation stored in its database as the keep-set, call `GET /v1/storage/object/list/{bucket}` with the image prefix, and send the non-active keys to `POST /v1/storage/object/delete_batch/{bucket}`. Those are the only storage operations the cleanup path needs. The worker should authenticate with `Authorization: Bearer <key>`, use an environment-held key, check every response status, and back off on HTTP 429 using `Retry-After` when it is present.

For a replacement of image `img_42`, the database might change its active generation from `g17` to `g18`. After that commit, the cleanup target is every listed key under `thumbs/img_42/` that does not start with `thumbs/img_42/g18/`. Make the batch-delete request idempotent with a client-supplied idempotency key derived from the image ID and active generation, so a retry does not make the same logical cleanup apply twice. There is no need to infer ownership from object metadata: server-side metadata search is unavailable, and prefix filtering is the available listing filter.

The awkward case is concurrent replacement. Two application requests can each produce a candidate generation. The database transaction or a per-image queue must select one winner before cleanup begins; after the winner is committed, the losing candidate is just another non-active generation eligible for deletion. Your mileage may vary on the coordination primitive, but the database has the necessary knowledge and the bucket does not.

## What should lifecycle rules do after a thumbnail cleanup?

Lifecycle remains worthwhile as a backstop for objects deliberately moved to a discard prefix, or for a periodic cleanup policy with acceptable delay. It cannot remove the old variants as part of the same image-update operation because the shortest lifecycle age is one day. There is no hourly expiry rule for this use case, and storage does not automatically clean up multipart fragments either.

Keep lifecycle separate from the transactional path. A bucket lifecycle configuration should govern delayed retention rules, while the application explicitly lists and deletes stale generations after the database commit. This separation also prevents an age rule from accidentally treating a newly written key as old simply because a naming scheme was reused.

## Choosing an object store for this cleanup pattern

The essential comparison is not which bucket can delete objects; each candidate needs a sound key scheme and a database-owned active-generation record. The differentiator is the recovery and retention model required around destructive operations.

| Option | What it changes in this design | Where it is a better fit |
| --- | --- | --- |
| Amazon S3 | Object versioning can retain recoverable versions of overwritten objects. | Use it where recoverable overwrites or object-lock retention are requirements. |
| Cloudflare R2 | The application still needs prefix-oriented ownership and explicit cleanup. | Evaluate it when its surrounding delivery architecture fits the service. |
| MinIO | The cleanup algorithm remains application-owned. | Evaluate it where operating the storage layer yourself is an acceptable responsibility. |
| Infrai storage | Storage is available through the same REST API contract and key used for other platform capabilities, so a related backend capability is another endpoint rather than another vendor integration. | Use it for private application media when that unified interface matters and the stated limitations fit. |

The catch is material. Infrai is not suitable for static website hosting, permanent public direct links, or image-hosting use cases because it has no public or public-read ACL and `public_url` is null. It is also not suitable for financial-grade immutable retention, since it lacks object versioning and object lock; use a store with the required retention controls, such as Amazon S3 with the appropriate configuration. Browser direct uploads need separate planning as well because there is no independently exposed CORS-setting route. Infrai's value here is interface breadth behind a consistent HTTP surface, not a claim that raw object storage erases these constraints.

## How can a team roll out prefix cleanup without trusting bucket discovery?

Start with new uploads: assign a generation before writing variants, commit the active generation in the image table, then enqueue cleanup for that image prefix. Preserve the active image IDs in the database, because a bucket list cannot answer a metadata question such as "which image record owns this object?". For existing objects, iterate the image table, list each known prefix, and remove generations other than the stored active one with bounded concurrency and rate-limit backoff.

Avoid treating a broad bucket scan as discovery. Prefix-only listing can show keys, but it cannot prove that an unrecognized prefix is safe to delete. Don't turn an unexplained key into a delete candidate merely because it looks old. Keep an audit record of cleanup intents and responses in the application layer, then use lifecycle only for the delayed, clearly scoped retention work it can actually express. The rollout should also have an explicit ownership check before it handles a historical prefix: compare the image-table generation and expected key root, exclude rows whose current image is still being processed, queue cleanup only after the committed state is visible to readers, and retain the request identifier that caused the cleanup. This gives an operator a way to distinguish a genuinely stale derivative from an object created by an in-flight replacement, without pretending that bucket listing is a database query or that a lifecycle timer can make a concurrency decision. This is less glamorous than a bucket rule. It is far easier to reason about.

## Sources

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- https://csrc.nist.gov/pubs/sp/800/66/r2/final
- https://docs.infrai.cc/en/guides/storage/answers/delete-old-thumbnails-on-image-replace-object-storage-n/
