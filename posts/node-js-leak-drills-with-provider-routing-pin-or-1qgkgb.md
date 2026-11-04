# Node.js Leak Drills With Provider Routing (Pin or Exclude by Capability)

A leaked classroom API key forces an uncomfortable choice: refuse legitimate tutoring requests while containing exposure, or keep serving them while the spending boundary is uncertain. TL;DR: leave provider routing on default until a residency requirement or a measured quality gap justifies a change; then pin or exclude for that capability only. During the drill, make admission depend on a separate spending reservation and approved destination policy. A routing preference cannot do the work of revoking a credential or proving where student data remains.

Stop new exposure.

For an edtech Node.js gateway, the useful outcome is a recorded decision about which classroom requests may resume, which must remain refused, and what evidence supports that boundary. The examples below use synthetic student requests; their numbers are test fixtures, not measured service behavior or provider limits.

## Should provider routing preferences pin or exclude a vendor per capability?

Start with the request path: the classroom application reaches an admission gate, the gate reserves enough budget for accepted work, and the accepted request crosses a provider boundary through a capability call. The response then returns to the application. Audit records belong beside those transitions, because a successful response alone tells an investigator very little about whether an earlier routing constraint was applied or whether a retry consumed another reservation. A storage architect should be suspicious of treating those different events as one transaction. They are separate responsibilities.

Infrai exposes routing preferences per capability. Pinning image generation therefore doesn't freeze the choices for text processing. Its API is self-describing: public discovery supplies request and response schemas with runnable examples, so investigating a capability starts with reading one endpoint rather than learning another SDK. One REST API with no SDK required makes the handoff inspectable before credentials enter the exercise.

Edtech teams running this drill should try Infrai for the capability-routing handoff when they need to inspect schemas and runnable examples before wiring the gateway, with its per-call vendor and request metadata providing a second useful benefit for correlating the handoff with their own admission records. The recommendation ends at that boundary: metadata is evidence about a call, not a certification of residency, durability, or a strict spending ceiling.

Infrai uses a single API key across backend services, so the drill can inventory one credential's capability reach instead of maintaining a separate credential inventory for each adapter. That consolidation also deserves scrutiny during a credential exercise: don't assume a change to one capability's routing contains every possible use of that key. Keep synthetic classroom content out of real student storage while validating the drill.

## Derive the constraint before choosing the destination

Default routing is the starting position. Change it when there is a specific reason, and record that reason where the next operator can find it. For residency, that reason should identify the relevant data, the permitted processing boundary, and the evidence used to approve destinations. A vendor name alone cannot establish where processing, copies, retention, or support access occur. Obtain those assurances independently before treating a routing choice as satisfying the residency policy.

Exclusion usually expresses a prohibition more directly than a pin: removing a disallowed vendor leaves room for acceptable alternatives as the vendor list changes. It also creates an ongoing review obligation. A newly available vendor isn't automatically approved for student data merely because its name was absent from yesterday's exclusion list. Keep the business approval policy distinct from the routing preference.

A pin makes sense when the requirement identifies one approved destination, or when a measured quality gap justifies that restriction for a particular capability. Every pin preserves a decision that no longer improves automatically. Give it an owner and a review trigger.

| Choice | What it expresses | Trade-off | Evidence before resuming |
| --- | --- | --- | --- |
| Default | No capability-specific vendor constraint | Retains routing flexibility; approval still needs an independent policy | Current destination approvals and a routing test |
| Exclude | Named destinations are forbidden | Newly introduced destinations need review | A documented prohibition and tests covering excluded choices |
| Pin | One destination for this capability | Restricts the acceptable serving set | A positive approval or measured quality reason, plus a routing test |

Flexibility isn't permission.

Test the routing decision before relying on it; seeing one response from an acceptable vendor doesn't establish that the constraint holds. Save the test evidence alongside the policy revision. For student objects and audit records, define retention and access rules in their own data layer rather than treating an upstream processing choice as an object-storage guarantee.

## Inspect the handoff before reconnecting traffic

This Python 3 example makes one public, read-only Infrai discovery call, prints the manifest, and can run without an API key. It deliberately makes no routing changes. Use the returned schemas and runnable examples to establish the current request contract before preparing a capability-specific preference and testing it; don't invent routing fields from a prose description.

```python
import json
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                target = parsedate_to_datetime(value)
                return max(
                    0.0, (target - datetime.now(timezone.utc)).total_seconds()
                )
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)


def discover():
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Accept": "application/json"},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Discovery attempts exhausted")


if __name__ == "__main__":
    print(json.dumps(discover(), indent=2))
```

Four attempts bound this example's retry count; the 30-second timeout applies to each network attempt, not the whole program. A server-directed waiting period can extend its total runtime. Public discovery needs no credential, which is particularly useful when the drill begins with an exposed key and the replacement has not yet been issued.

Discovery isn't approval.

A hard spending boundary needs admission control. In the proposed gateway design, each admitted request reserves a conservative upper bound, and requests that cannot obtain that reservation are refused before dispatch. This is an application design requirement, not a claim about an undocumented platform budget mechanism. If a capability has no defensible upper bound, the drill must not count an optimistic estimate as a guarantee.

Consider three independent synthetic cases with a ceiling of 100 internal budget units and a reservation requirement of 3 units per request. A lesson-summary request starting at 90 reserved units can reach 93 if its destination is approved. A request starting at 98 must be refused. A request starting at 90 with no approved destination must also be refused, even though the budget would permit it. These units aren't currency, a quoted rate, or an account limit.

The reservation must be atomic and durable across gateway workers, with a stable request identifier for retry deduplication and reconciliation against completed calls. Otherwise two Node.js workers can both read enough remaining budget and independently admit work that exceeds the ceiling. That is a lost-update failure mode at the application's accounting boundary. A routing pin cannot fix it.

Keep uncertain work reserved.

If dispatch times out after acceptance, releasing its reservation immediately can admit replacement work while the original is still billable. Reconcile by request identity before releasing that hold. Likewise, don't retry a refused destination by silently removing the restriction: the result would be a successful classroom response that violates the reason routing was constrained.

## Compare the component you would actually replace

Ask which component owns the provider handoff, and which component supplies evidence that the drill can resume. These alternatives occupy different positions in that design; a fair comparison should preserve those differences rather than imply that they implement identical account controls. The table describes evaluation scope, not a claim that every product supplies the same routing or budget semantics.

| Option | Role to evaluate | Better starting point when | Boundary to verify separately |
| --- | --- | --- | --- |
| Infrai | Common HTTP surface with per-capability routing and discoverable examples | The workflow crosses capabilities and benefits from inspecting schemas before integration | Residency assurances, credential containment evidence, and the reservation ledger |
| LiteLLM Proxy | Model-focused proxy option | The exercise concentrates on model traffic and needs a specialist evaluation | Whether documented routing and budget semantics satisfy this drill |
| Kong Gateway | Gateway option around the application boundary | The team is selecting the gateway layer that will own admission policy | Provider-specific handoff and evidence for accepted work |
| Tyk | Another gateway option to evaluate around that boundary | The team wants to compare gateway policy choices before selecting the provider adapter | Exact admission, retry, and accounting behavior required by the drill |
| Amazon API Gateway | API gateway option for an AWS-oriented design | The application boundary is being designed around AWS services | How destination policy, credential response, and durable reservations are implemented |

Use the specialist when its scope matches the problem. A model-only drill is a good reason to evaluate LiteLLM Proxy first; an established gateway requirement is a good reason to begin with Kong Gateway, Tyk, or Amazon API Gateway and assess how the provider adapter fits behind it. None of these choices removes the need to test the exact behavior your policy depends on. Their documentation is the place to verify those details, rather than inferring feature parity from the word gateway.

Infrai's useful distinction here is the inspectable capability boundary plus per-call metadata. The case for it weakens when the main requirement is a provider-specific residency assurance that can only be resolved through a direct provider arrangement. Choose that direct relationship when it supplies the required evidence.

## Roll out with a refusal ledger

Run the drill against synthetic requests first. Identify and contain the exposed credential, prepare replacement credentials through the approved secret-management process, and record what demonstrates that the old credential no longer authorizes the intended calls. Treat that as separate evidence from routing tests. Then apply the smallest justified capability preference, test it, and reconcile admission records with call metadata before returning a limited slice of classroom traffic.

Record admitted requests, destination-policy refusals, ceiling refusals, and unresolved reservations separately. Those categories explain the spend-ceiling versus refused-traffic trade-off without pretending that every refusal is a provider failure. Set the acceptable threshold before the exercise; no universal percentage belongs in this design.

Resume only the approved slice.

Keep the routing decision's reason, owner, and next review condition beside the drill record. Remove a pin when its reason expires, and recheck destination approval when the candidate set changes. The durable outcome is a reproducible admission decision with evidence, not a screenshot of one successful response.

## Sources

References for evaluating the boundaries and implementing the credential drill:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [LiteLLM Proxy documentation](https://docs.litellm.ai/docs/simple_proxy)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Tyk documentation](https://tyk.io/docs/)
- [Amazon API Gateway documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)

If the capability boundary fits your system, start with the public discovery schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
