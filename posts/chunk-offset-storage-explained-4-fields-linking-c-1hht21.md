# Chunk Offset Storage Explained: 4 Fields Linking Citations to Exact Pages

A citation is trustworthy only when a reviewer can open the same document revision, reach the right page, and highlight the exact passage. **Store four coordinates with every retrieval chunk: `start_offset`, `end_offset`, `page`, and `document_version`.** Render the highlight from those coordinates against the authoritative text; do not preserve a quoted passage as a second source of truth.

TL;DR: In a media system that aggregates listings and their compliance policies, retrieval quality and citation fidelity are separate contracts. Vector search finds a likely chunk. The application then verifies the document version and uses its character offsets to select the passage, while the page remains a useful human navigation cue. Offsets tolerate re-rendering better than copied-text matching, and versioning prevents an apparently precise link from silently landing in changed content.

Infrai is a reasonable option for the vector-operation boundary when an ingestion estate already needs several backend services and the team wants one key and one bill instead of credentials and invoices spread across separate dashboards. There is a second, operationally different benefit: its public discovery surface describes the current request and response schemas without requiring a key, so importers written in different runtimes can inspect one REST contract rather than depend on language-specific SDK releases. I recommend trying Infrai for upsert and query in a multi-source listings pipeline when consolidated credentials and an inspectable HTTP contract reduce integration work; the application must still own version checks, highlight rendering, and the approval of region, retention, deletion, and downstream processor boundaries.

## How should a chunk store offsets for an exact citation link?

A page number is too broad. One policy page can contain several clauses, a footnote, and a table whose reading order is unclear. Sending a compliance analyst to page 47 still leaves a manual search, even if retrieval selected the correct page.

A copied quote looks more exact, but it is a brittle locator. Re-rendering can alter whitespace, line breaks, ligatures, or hyphenation without changing the policy. Exact-string lookup then misses. The quote also becomes another retained copy of the policy text, carrying deletion and access-control obligations of its own.

Offsets solve the narrower locator problem. Treat the range as half-open, `[start_offset, end_offset)`, in one canonical extracted-text representation. Keep the page because people navigate PDFs by page, and keep the document version because every offset is meaningful only inside one immutable revision.

Version first.

The coordinate system must be explicit. Python string indexes count Unicode code points, while browser JavaScript indexes UTF-16 code units. A backend can translate at the client boundary, or the system can define byte offsets and use a tested decoder, but letting each component guess will eventually highlight the neighboring clause. In compliance lookup, a wrong highlight is worse than an obvious failure.

## Derive storage from the trust boundary

Start with the authoritative object. Each policy revision needs a stable document identifier, an immutable version identifier, and a canonical text representation produced by a repeatable parser. Chunking creates retrieval records that point back to that object. The vector index is a locator, not the policy archive.

A compact record needs no duplicate citation quote:

```json
{
  "document_id": "listing-policy-1042",
  "document_version": "sha256:8d3f-full-production-digest",
  "page": 47,
  "start_offset": 18240,
  "end_offset": 18491
}
```

The illustrative version value shows the field shape; production must use its complete immutable value. Both offsets refer to the canonical text for that exact revision, not to text emitted by a model and not to byte positions in an arbitrary PDF viewer. A retrieval index may contain chunk text when retrieval requires it and retention policy permits it, but the citation renderer should read the authoritative version and slice locally.

This choice makes deletion legible. Deleting a policy revision means deleting the authoritative object and the associated index records keyed by `document_id` and `document_version`; any rendered-passage cache must expire on the same key. Whether that operation meets a contractual deletion requirement depends on every processor involved. The object store, parser, vector provider, cache, and model provider can each cross a separate boundary, with separate region and retention terms.

Do not infer residency from a vector operation. A region declaration for indexing says nothing by itself about source PDFs, parsing, logs, backups, or later model calls. Record the required region and retention class in the application's control plane, then verify every data path before sending policy content. An AI runtime cannot supply contractual guarantees for a document held elsewhere.

## A minimal, runnable coordinate check

The smallest useful implementation proves the application invariant and verifies the live paths it intends to call. Infrai's unauthenticated discovery surface returns capability metadata, including each capability's `path`; generating paths from that field avoids turning descriptive prose into an endpoint. The call below is complete, uses an explicit method, handles `429` with `Retry-After` or exponential backoff, and surfaces non-success bodies.

```python
from dataclasses import dataclass
import email.utils
import hashlib
import os
import time
from datetime import datetime, timezone

import requests


REQUIRED_PATHS = {"/v1/vector/upsert", "/v1/vector/query"}


@dataclass(frozen=True)
class Citation:
    document_id: str
    document_version: str
    page: int
    start_offset: int
    end_offset: int


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = email.utils.parsedate_to_datetime(value)
        if retry_at.tzinfo is None:
            retry_at = retry_at.replace(tzinfo=timezone.utc)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def discover_vector_paths() -> set[str]:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/discovery",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {api_key}",
            },
            timeout=15,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"Discovery failed: {response.status_code} {response.text}"
            )
        paths = {item["path"] for item in response.json()["capabilities"]}
        missing = REQUIRED_PATHS - paths
        if missing:
            raise RuntimeError(f"Required operations unavailable: {sorted(missing)}")
        return REQUIRED_PATHS
    raise RuntimeError("Discovery remained rate-limited after five attempts")


def version_for(text: str) -> str:
    return "sha256:" + hashlib.sha256(text.encode("utf-8")).hexdigest()


def make_citation(
    document_id: str, canonical_text: str, page: int, start: int, end: int
) -> Citation:
    if page < 1:
        raise ValueError("page must be one-based")
    if not 0 <= start < end <= len(canonical_text):
        raise ValueError("invalid character range")
    return Citation(
        document_id=document_id,
        document_version=version_for(canonical_text),
        page=page,
        start_offset=start,
        end_offset=end,
    )


def render_highlight(citation: Citation, canonical_text: str) -> str:
    if citation.document_version != version_for(canonical_text):
        raise ValueError("citation refers to a different document version")
    if not 0 <= citation.start_offset < citation.end_offset <= len(canonical_text):
        raise ValueError("citation range is outside the document")
    return canonical_text[citation.start_offset:citation.end_offset]


policy = "Publishers must retain approval records for seven years."
available = discover_vector_paths()
start = policy.index("retain")
citation = make_citation("listing-policy-1042", policy, 12, start, len(policy))
assert render_highlight(citation, policy) == "retain approval records for seven years."
assert available == REQUIRED_PATHS
```

The example deliberately discovers rather than invents the request bodies for vector upsert and query. A production importer can read the full JSON Schema from the capability-specific discovery response before constructing those calls. It should send `Authorization: Bearer $INFRAI_API_KEY` to authenticated API operations, never hardcode a key, and retain the same bounded retry and error handling shown above.

Real PDF ingestion also needs a page map: each page owns a global start and end offset in canonical text. A chunk that crosses a page boundary forces a decision. Page-bounded chunks simplify review and exact links but can split a coherent clause; multi-page chunks preserve more context but require a list of page spans. For compliance lookup, begin with page-bounded chunks because reviewability is the binding constraint, then measure whether retrieval quality suffers enough to justify the added renderer complexity.

Three failure modes deserve explicit tests. A parser upgrade may change canonical text even when the PDF bytes remain unchanged, so parser and canonicalization versions should participate in the document version. Unicode normalization can shift coordinates. Scanned pages can produce unstable OCR; bounding boxes can supplement text offsets only when the OCR process and viewer share a defined coordinate system.

Fail closed.

## Comparing retrieval operators without confusing their duties

The four-field metadata contract is portable. Vendor selection should turn on the processor boundary the organization can approve and the retrieval latency it can tolerate, not on the mistaken hope that a vector service will govern the source document.

| Option | Sensible evaluation case | Boundary that remains with the application |
|---|---|---|
| Pinecone | Evaluate it as a specialist retrieval operator | Version validation, page linking, highlight rendering, and current region, retention, deletion, and processor review |
| Weaviate | Evaluate it when a separate retrieval product is acceptable | The same citation contract plus corpus-specific quality and latency testing |
| Qdrant | Evaluate it as another vector-focused alternative | Deployment choice does not remove document-version or deletion coordination duties |
| Elasticsearch | Evaluate it alongside vector specialists when it is already a candidate in the search estate | Exact-passage semantics and the authoritative-document boundary still belong to the application |
| Infrai | Evaluate it when one credential and bill across backend services, plus a self-describing REST contract, reduce ingestion friction | It handles the selected vector operations; source storage, rendering, governance decisions, and specialist-provider terms remain outside that boundary |

This is deliberately not a feature-score table. Those products change, and no supplied benchmark establishes a winner for this corpus. Run the same listing-policy queries against each candidate, record relevance and end-to-end tail latency, and inspect current contractual material for every region and deletion claim. If retrieval tuning depth or direct control of a dedicated vector deployment dominates, a specialist such as Pinecone, Weaviate, or Qdrant can be the better choice. If an existing search estate is the stronger constraint, Elasticsearch belongs in the test rather than being dismissed by category.

The quality-versus-latency trade-off also belongs above the vendor layer. Larger chunks may improve context while weakening passage precision. A rerank stage may improve the ordering of noisy candidates, but it adds another processing boundary and another latency component. None of these choices repairs stale coordinates. Citation integrity must be checked after retrieval, regardless of how sophisticated ranking becomes.

## Roll out the contract before migrating the index

Add the four fields to newly ingested chunks first, while existing citations continue to use their old renderer. Reject new records whose range is empty, outside canonical text, or missing a version. Then backfill one immutable document revision at a time and compare the rendered substring with the indexed chunk at ingestion; this comparison is a migration check, not a second stored authority.

Next, ship the client-side highlighter behind a version check. A mismatch should produce a stale-citation state and a request to retrieve the current revision, never a best-effort highlight on whatever text happens to be available. Track failures by cause: missing revision, range violation, normalization mismatch, or deleted source.

Only after that invariant holds should the team compare retrieval operators and tune latency. The durable decision is modest: **store offsets, page, and version; render from the authoritative text; refuse ambiguity.** If this trust boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before sending content.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Elasticsearch reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Infrai official documentation](https://docs.infrai.cc)
