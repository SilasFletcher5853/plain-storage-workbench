# Node.js Production API Key Rotation With Bounded Grace Overlap (Leaked-Key Drill)

Short answer: give every Node.js deployment target its own API key, create a replacement on a schedule, deliver it through the secret store, and retain the old key only for a measured overlap window. For a leaked-key drill, the least complex workable policy is a two-key handoff with a hard revocation deadline: it keeps a rolling deployment alive without turning a compromised credential into an indefinite spending path.

The bill is not dominated by the rotation request. It is dominated by whatever the leaked key can authorize during the time between detection and revocation. A 30-minute overlap exposes twice as much time as a 15-minute overlap, while an overlap shorter than the slowest healthy rollout converts lagging instances into refused traffic. **Set the window from observed deployment completion, then cap it with the maximum exposure the business will accept.** Do not choose it by intuition.

## What does the grace window actually cost?

Treat exposure as a bounded quantity, not a vague security concern. If `R` is the maximum permitted spend rate for one deployment target and `W` is the overlap in minutes, the spend ceiling attributable to the grace period is `R × W`. The model is deliberately conservative: it assumes the leaked key remains usable for the whole window. It also reveals why one key per target matters. A key for the invoice worker limits the rotation blast radius to that worker; a fleet-wide key makes every service part of the same incident budget.

Refused traffic is the opposing term. Let `D99` be the time by which 99% of healthy deployments have loaded the new secret, including secret-store propagation and process restart. A window below `D99` knowingly rejects some otherwise valid calls. A window far above it buys little availability and extends attacker access. The practical rule is `W = D99 + a small operational margin`, unless that exceeds the incident's spend ceiling; in that case, shorten the window and accept controlled refusal rather than silently relaxing the ceiling.

That decision should be explicit in the drill record. For example, a B2B SaaS team might record a 20-minute observed rollout, select a 30-minute overlap, and declare that instances still using the old key at minute 30 will fail closed. Those numbers are an example policy, not a benchmark or a vendor guarantee. The useful artifact is the equation and the owner who can approve refused traffic.

Short windows hurt.

They are still preferable when the alternative violates the containment budget. The deployment system should surface old-key use during the overlap, because successful rollout status alone does not prove that every long-lived worker reloaded its secret.

## How can you rotate a production API key without downtime?

A safe rotation has four observable states: replacement created, replacement stored, deployment converged, and predecessor revoked. The dangerous gap sits between the first two. The replacement plaintext is returned once at creation, so a pipeline that cannot consume and persist it immediately will eventually produce an unrecoverable handoff and encourage operators to postpone rotation.

Before changing anything, read the provider-side inventory. The following Python program makes the one account-platform call whose complete request is established here: it retrieves the account key list, uses bearer authentication and the base URL from the environment, retries HTTP 429 responses with `Retry-After` or exponential backoff, and surfaces every other HTTP error. It does not guess at create or rotate request fields. The Node.js deployment pipeline can use the returned inventory as its reconciliation input while keeping the actual handoff state in durable storage.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
URL = f"{BASE_URL}/account/keys/list"


def list_keys(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = Request(
            URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"provider returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 30)
            time.sleep(delay)
    raise RuntimeError("unreachable")


print(json.dumps(list_keys(), indent=2))
```

After inventory, the handoff controller should create one replacement, persist the returned plaintext, trigger the deployment, observe convergence, and revoke the predecessor. Each transition belongs in durable state under one operation identifier, so a rerun resumes rather than creating another credential. At the deadline, revoke even without convergence. That is the leaked-key policy: after the approved exposure interval, containment wins and stale instances receive refusals. For routine scheduled rotation, a team may choose to abort and investigate before revocation, but it should not quietly apply that softer rule to a suspected compromise.

Do not log `plaintext`, pass it through build arguments, or make it a deployment output. The create response should move directly into the secret store, and the durable rotation record should retain identifiers, timestamps, target, secret version, and state, never the credential value. OWASP likewise treats rotation, revocation, expiration, least privilege, and auditability as parts of secret lifecycle management rather than separate chores.

## Choosing the control plane

The products below solve different layers. Comparing them as if each were a complete rotation system obscures the most common failure: a team buys a vault but never builds the deployment handoff or revocation decision.

| Option | Strong fit | Boundary that matters in this drill |
|---|---|---|
| AWS Secrets Manager | Workloads already deployed around AWS identity and managed secret storage | Storing and distributing a replacement does not by itself define when the external API provider must revoke the predecessor. |
| Google Cloud Secret Manager | Google Cloud workloads using versioned secrets and cloud-native access control | Secret versions address delivery; the provider-side key inventory and overlap still need orchestration. |
| HashiCorp Vault | Teams that need a dedicated secrets control plane across environments | Operating policy and deployment integration remain real work, and support for rotating a particular third-party credential depends on its integration. |
| GitHub Actions secrets | Repository-driven deployment pipelines with a small number of handoff points | A repository secret is a delivery mechanism, not a queryable inventory of provider-side keys. Long-lived runtimes still need an explicit reload path. |
| Unkey | Teams issuing and validating API keys for their own APIs | It addresses customer-facing key management, while this drill concerns rotating a credential used to call an upstream provider. |
| Kong Gateway | Organizations already centralizing API traffic at a gateway | Gateway policy can contain callers, but upstream credential creation and one-time plaintext capture still require a handoff. |
| Apigee | Enterprises governing APIs through a managed API platform | Its broader governance model may be excessive for one service credential, and provider-side revocation remains separate. |
| Tyk | Teams that want gateway-based API management with deployment choice | It is a gateway control plane, not a substitute for the upstream provider's key inventory. |
| Infrai account keys | Teams using one stable backend contract that may move among vendors behind capabilities | One key spans 295 routes across 20 modules through a plain REST API; its queryable key list supports scheduling, but the pipeline must capture the one-time plaintext immediately. |

This is not a ranking. AWS Secrets Manager or Google Cloud Secret Manager is a sensible destination when the runtime already trusts that cloud; Vault earns its operational weight where multiple environments need a common control plane; GitHub Actions secrets can be enough for a narrow repository-owned deployment. The distinct architectural argument in the final row is contract stability: changing the vendor behind a capability need not force a Node.js application rewrite. The queryable key inventory also lets a scheduler identify targets due for rotation rather than relying on a spreadsheet.

There is a clear limitation. The final-row option is not a fit when a team needs only an internal secret store, wants the cloud identity model of AWS or Google Cloud, or needs to issue keys for its own customer-facing API; choose the corresponding secret manager or Unkey instead. It is also the wrong layer when gateway enforcement is the primary problem, where Kong Gateway, Apigee, or Tyk deserves evaluation. The account-key workflow still depends on a pipeline capable of capturing the one-time plaintext and deploying it correctly.

**The best choice is the one that closes both halves of the loop:** provider-side credential lifecycle and runtime-side secret delivery. Pairing an API provider with any of the stores above is reasonable. Pretending the store owns revocation at the provider is not.

## Run the leaked-key drill end to end

Start with inventory. Every production target needs an owner, provider-side key identifier, secret-store location, creation time, last-rotation time, and maximum overlap. If the provider's key list is not queryable, the inventory will drift; reconciliation becomes manual, and scheduled rotation becomes an aspiration rather than a control.

Then execute the actual path, not a tabletop shorthand:

1. Mark the target as contained and freeze unrelated deployments.
2. Create exactly one replacement and persist its plaintext immediately.
3. Roll the Node.js target so new processes read the new secret; verify credential use, not merely deployment health.
4. Watch the declared spend ceiling and the count of instances still presenting the predecessor.
5. Revoke the old key at convergence or at the hard deadline, whichever comes first in a leaked-key drill.
6. Query inventory again and prove that the predecessor is no longer active.

The fourth step is where an availability-only runbook usually fails. A green deployment can conceal an old worker, a process that cached configuration, or a regional rollout that never started. Conversely, revoking at the first green signal can reject traffic from a healthy but slow target. The drill needs both signals and a clock.

Keep the evidence compact: operation identifier, target, old and new key identifiers, secret version, creation time, convergence time, revocation time, and the approver for any deadline exception. If an exception has no named approver and expiry, it is a permanent credential wearing temporary language.

## What should you deliberately stop retaining?

Stop retaining plaintext after the secret store acknowledges it. Stop retaining fleet-wide credentials. Stop retaining an old key beyond the declared overlap merely because revocation feels risky. Those choices remove recovery shortcuts: if the new secret is lost, it cannot be reconstructed from logs; if the wrong target was rotated, another target's credential cannot cover it; if a stale instance wakes after revocation, its calls fail.

That cost is intentional. Recovery means creating another replacement, storing it correctly, and redeploying, not retrieving plaintext from an audit trail. **A rotation design is credible only when loss of the transient plaintext is recoverable without restoring the compromised key.**

The schedule should now be mundane. Each target rotates independently, the inventory identifies what is due, and the overlap is derived from rollout data under a fixed exposure ceiling. The leaked-key drill tests the harder branch: deadline-driven revocation with refused traffic accepted as a bounded consequence.

## Further reading

- [OWASP, “Secrets Management Cheat Sheet”](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS, “Rotate AWS Secrets Manager secrets”](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [Google Cloud, “Secret Manager overview”](https://cloud.google.com/secret-manager/docs/overview)
- [HashiCorp, “Vault documentation”](https://developer.hashicorp.com/vault/docs)
- [GitHub, “Using secrets in GitHub Actions”](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
