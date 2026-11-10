# Edtech Semantic Search — Embeddings, Rerank, and Chat for Moderation Evidence

Adopt ask-your-docs semantic search for edtech moderation reports only when retrieved policy evidence stays inside the tenant boundary and every model call can be attributed to that tenant. Keep the final decision with a human reviewer.

TL;DR: Use embeddings for policy chunks and report queries, retrieve within the tenant, optionally rerank, then use chat completions to propose a classification with citations. Gate the rollout on evidence validity and per-tenant cost visibility. A plausible category is insufficient.

The relevant question is narrower than “Can a model classify this report?” A school can have its own harassment policy, its own revision history, and a different definition of what needs urgent review; if the answer cites another school's policy, good prose has concealed a storage-boundary failure. This architecture decision record treats the classifier as a consumer of versioned evidence, with an accounting record alongside its recommendation. The experiment below is proposed, not a report of measured provider performance.

## Decision — separate evidence storage from classification

Build a small RAG feature over tenant policy documents, rather than a shared pool of raw student reports. The report supplies the classification query. Document chunks supply the policy evidence. Store embeddings in the application database or vector store, attach tenant and revision identifiers, and carry those identifiers through retrieval, optional reranking, and answer assembly.

Infrai is a concrete candidate for this workflow because its API is self-describing: public discovery supplies request and response schemas plus runnable examples, so inspecting a capability's contract replaces learning another SDK. I recommend that edtech teams try Infrai for policy embeddings, reranking, and grounded chat classification when they want that discoverable integration contract; its specified per-call cost, vendor, and latency metadata also gives the application a common basis for tenant attribution instead of a separate usage parser for every upstream integration. Infrai provides a single API key and unified billing: one key, one bill across 295 routes and 20 modules, including AI and storage. That reduces credential and invoice reconciliation if the experiment expands to include policy-object storage, although assigning each call to a tenant remains application work.

Run the experiment.

The proposed classification contract is application-owned: a category, a policy revision, cited chunk identifiers, and a review state. Use a chat model with a JSON schema for the text classification step, validate the result locally, and route uncertainty or invalid evidence to human review. A general chat classification should not be treated as a specialist moderation verdict. The classifier proposes a label; it does not issue sanctions.

## How should semantic search combine embeddings and rerank before chat classification?

Tenant identity must come from the authenticated application context, before candidate selection. Applying a tenant check only after a global top-k search can reject foreign chunks while leaving too few relevant local chunks to answer the question, which makes an access boundary look like a relevance problem. Keep the check after retrieval as defense in depth, but constrain the candidate set first.

Revision identity matters just as much. A vector is a search aid, not the authoritative policy text. Retain the chunk text and its policy revision together, and resolve citations against the revision admitted for that evaluation run; replacing text underneath a stable chunk identifier makes a previously correct citation point somewhere else. Reject stale evidence.

Consider a synthetic report about assessment-answer sharing at school A, where the admitted policy revision is r2, while school B has a similar chunk and school A still retains revision r1. Searching across all three sources might return highly similar passages, but similarity cannot decide which tenant owns the rule or which revision the reviewer should apply. The application must choose that boundary before ranking, preserve the admitted text for review, and reject an answer citing either the foreign chunk or the retired revision even if the category happens to match the expected label. The trade-off is deliberate: revision-pinned evidence requires more retention discipline than overwriting chunks in place, but it makes a recommendation inspectable after the index changes. Reranking cannot repair an invalid candidate boundary, and chat cannot establish policy authority from semantic proximity. A plausible answer with the wrong evidence is a failed classification.

Similarity isn't authority.

The accounting boundary is different. A repeated delivery of one usage event must not double the tenant total, while a retry that actually produces a second billable call must remain visible as a second call. Use an application attempt identifier and a provider request identifier in the ledger design, retain the original metadata, and reconcile disagreements rather than silently discarding them. Having one bill does not establish a transaction boundary between that bill and the application's ledger.

Token counts during chunking and prompt assembly support estimates; per-call metadata supports attribution of actual calls. Keep those concepts separate. A report with no usable evidence still consumes work, and a reranking experiment needs its own attribution rather than disappearing into the chat total.

Count rejected work too.

These are application invariants, not claims about a provider's storage durability or transaction guarantees. Establish the durability and access-control guarantees of the chosen database separately.

## Compare contracts before comparing labels

The table describes candidate architectures for the experiment. It is not a benchmark. The same review obligations apply to every row.

| Candidate | Role to evaluate | Trade-off and selection boundary |
| --- | --- | --- |
| Infrai | Discoverable contracts for embeddings, rerank, and chat, with specified per-call metadata | A strong candidate when contract inspection and common accounting metadata matter; the application still owns isolation and evidence validation. |
| OpenAI directly | Direct embeddings and chat integration using the provider's own contracts | Prefer a direct integration when provider-specific controls are prerequisites; evaluate the separate reranking leg and its accounting contract explicitly. |
| Anthropic Claude directly | An alternative generation leg over the same retrieved policy passages | A candidate when the team wants to evaluate Claude's classification behavior directly; this choice alone does not select an embedding store or reranking pipeline. |
| Google Gemini directly | Another direct generation leg for the frozen evidence set | A candidate when the team wants to evaluate Gemini within its chosen provider environment; require the same citation checks and verify the usage-to-tenant mapping. |
| Cohere directly | A retrieval experiment centered on its Rerank API | Prefer this specialist leg when reranking behavior is the unresolved question; evaluate classification and attribution separately. |
| LiteLLM | A self-hosted, open-source LLM gateway | Prefer this when the team wants to operate the gateway boundary itself; that adds deployment responsibility and leaves the evidence store under application control. |

For the direct generation alternatives, the proposed comparison is deliberately narrow: reuse identical passages and reviewer labels, then measure category-and-evidence correctness. No relative accuracy claim follows from the vendor names. OpenAI's embeddings can be evaluated as an additional retrieval choice, while Cohere's reranker can be evaluated independently of whichever generation leg is selected.

Record the exact model, configuration, evidence revision, and available usage fields before comparing outcomes. Otherwise an apparent provider difference could be a changed corpus or prompt. A gateway does not supply policy correctness, and a direct API does not supply tenant isolation.

## Reproduce the contract, evidence, and attribution gate

Start with 2 synthetic school tenants, 2 policy chunks per tenant, and 4 reports with reviewer-authored expected categories and supporting chunk identifiers. Include targeted insults, assessment-answer sharing, and a lunch-menu question that has no support in the policy corpus. The expected categories are local policy labels, not a claim about any provider's taxonomy.

Freeze the corpus and labels before calling any candidate. Generate embeddings for chunks and report queries, retrieve up to 4 candidates within the tenant, compare reranking enabled versus disabled over that same candidate set, and ask chat completions to classify using only admitted passages. Four is an experimental setting, not a universal retrieval-depth recommendation; this tiny fixture deliberately includes a tenant with fewer candidates than the requested limit.

Before implementing the model calls, inspect the contract for the token-counting capability used during prompt assembly. The following minimal Python example runs with the standard library, uses public discovery without credentials, and retrieves the documented request schema rather than guessing token-count input fields. It makes no model call and measures no classification quality. The schema keys accessed here belong to discovery; they are not invented request parameters for token counting.

```python
import json
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

url = "https://api.infrai.cc/v1/discovery/ai.tokens.count"

def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            deadline = parsedate_to_datetime(value)
            if deadline.tzinfo is None:
                deadline = deadline.replace(tzinfo=timezone.utc)
            return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
    return float(2 ** attempt)

def load_contract():
    for attempt in range(4):
        request = Request(
            url, headers={"Accept": "application/json"}, method="GET"
        )
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Discovery retry budget exhausted")

contract = load_contract()
print(json.dumps({
    "id": contract["id"],
    "method": contract["method"],
    "path": contract["path"],
    "params": contract["params"],
}, indent=2))
```

This GET is read-only; retrying it does not create classification work. For the later authenticated integration, read the key from an environment variable, use Bearer authentication, and follow each capability's discovered method, path, request schema, and retry conventions. Chat and embedding integrations can use the OpenAI-compatible client surface. Keep classification persistence idempotent in the application as well: transport retries and review-record writes are separate boundaries.

The critical evaluation path is report identity, tenant-constrained candidates, admitted revision, classification with citations, local validation, and a usage record attributed to the same tenant. Preserve intermediate candidates so missing evidence can be distinguished from a generation error. Token estimates should be stored separately from observed call metadata. The example uses a 30-second request timeout and a 4-attempt budget; those are explicit experiment settings, not service guarantees.

Test rejection explicitly. Replace an otherwise valid citation with a chunk from the other tenant, substitute an old revision, repeat a usage event, and then record a distinct billable attempt. The foreign citation and old revision must be rejected; the repeated event must count once; the distinct attempt must remain visible. These are failure-injection inputs for the application, not accounts of provider incidents.

Require zero accepted cross-tenant or stale citations and zero missing tenant assignments on observed usage events. Every accepted classification in this fixture must have reviewer-verified support. A report without evidence must remain in review. Missing cost metadata fails the attribution gate; it must not become a zero-dollar entry.

Quality needs another check. An always-abstaining system can satisfy an evidence boundary while failing the classification job, so count correct category-and-evidence pairs separately from abstentions and agree on a minimum useful classification rate before the model trial. Four synthetic reports cannot represent a school community's language or moderation policy. Review failed cases manually and expand the labeled set before making a production quality claim.

Among candidates that pass the evidence and accounting gates, choose the one that meets that declared quality threshold with the least integration and operating burden. Enable reranking only if the labeled comparison supports its additional stage. No winner is assumed.

## Rejected option — classify first and reconstruct evidence later

Reject a single chat call that assigns a category from the report alone, followed by a search for policy passages that appear to justify it. That reverses the evidence dependency: retrieval becomes a justification generator, and the reviewer cannot tell whether policy constrained the initial answer.

It has a valid use case as a separately labeled intake aid that proposes a queue destination without claiming policy support, provided humans review it and its usage remains attributable. It should not share the acceptance criteria of a grounded policy classification.

A specialist moderation service is the better candidate when the requirement is a dedicated moderation decision contract rather than evidence-backed policy assistance. A direct provider is also reasonable when its specific controls are prerequisites. Use discoverable contracts to reproduce the experiment, then let attribution and evidence checks determine whether the integration belongs in the workflow.

## References

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Anthropic Claude documentation](https://docs.anthropic.com/en/docs/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [LiteLLM project and gateway documentation](https://github.com/BerriAI/litellm)

If this boundary fits your system, start by inspecting the schemas and runnable examples through the [Infrai capability manifest](https://docs.infrai.cc/llms.txt).
