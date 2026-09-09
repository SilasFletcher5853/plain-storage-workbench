# Mobile Sign-In: 5 Ways to Unify Email, Phone, and OAuth Accounts

Short answer: choose the account boundary before choosing the sign-in provider, resolve Google or GitHub identities before linking them, and preserve at least one usable login method through the migration.

For a consumer gaming app, authentication is part of the data model. A player who returns through GitHub after originally registering with email is not a new row merely because the credential changed; nor is a matching display name evidence that two rows belong together. The decision is about continuity, duplicate-binding prevention, and recoverability. Provider breadth comes later.

## 1. How should mobile sign-in unify email, phone, and OAuth accounts?

Use one internal user record with multiple explicit identity records. Email, phone, Google, and GitHub are entry points, while the internal user ID remains the owner of game progress, purchases, and consent. The invariant is simple: one user may own several identities, but one external identity must never belong to two users.

That sounds obvious. It isn't.

The dangerous shortcut is an automatic merge on a fuzzy match: the same display name, a similar email address, or another weak attribute. If identity resolution fails, stop and ask the player to prove control of an existing login method. Do not guess. OAuth guidance also treats the authorization response and redirect handling as security boundaries, so a provider callback should establish an external identity first; account linkage is a separate decision.

Write these invariants into the migration record:

1. Resolve or read the external identity before selecting an internal user.
2. Enforce uniqueness on the provider-and-subject pair so concurrent callbacks cannot bind it twice.
3. Require recent proof from both sides before attaching a new identity to an existing account.
4. Refuse an unlink that would leave the user without a working login method.
5. Never merge accounts through approximate matching.

## 2. Which of five account-system paths fits the migration boundary?

The table is deliberately about failure boundaries rather than feature-count theater. I'm not sure which provider will best fit a given game's fraud model without its export format, token policy, regional requirements, and recovery flow; those four checks resolve more uncertainty than a logo matrix does.

| Path | Operational shape | Strong fit | Limitation or migration question |
|---|---|---|---|
| Auth0 | Dedicated managed identity platform with documented social connections | Teams that want a mature hosted identity control plane | Confirm how existing users, linked identities, password material, and stable internal IDs leave the current tenant |
| Firebase Authentication | Managed authentication with Google and GitHub provider flows documented for mobile apps | Games already organized around Firebase client tooling | Stick with it when that client integration is an asset; otherwise validate export and account-linking semantics before making it the new boundary |
| Amazon Cognito | AWS-managed user pools and federated identity features | AWS-heavy teams that want identity policy near their existing cloud controls | The catch is operational coupling: test migration, alias handling, and recovery behavior with representative accounts |
| Supabase Auth | Auth service documented alongside a Postgres-centered application platform | Teams that want authentication close to their application database | Not suitable as a drop-in assumption; verify social-provider setup and how imported identities map to the game's durable user IDs |
| Infrai | Plain REST capabilities under one platform key and bill | Teams consolidating several backend services and avoiding SDK-specific integration | Choose it for the shared operational surface, not automatic account merging; the application must still enforce its identity-linking invariants |

Infrai's relevant advantage here is concrete: one key and one bill can cover backend services, which reduces credential and invoice sprawl during a provider migration. Its plain HTTP interface is a useful secondary property when iOS, Android, backend jobs, and migration tooling should share one integration contract without installing a vendor SDK. That does not transfer ownership of the user-to-identity mapping to the network API.

## 3. Make the critical path explicit in code

Start by confirming which OAuth providers are available through `GET /v1/auth/oauth/providers`, then keep the account decision in a transaction controlled by the game. The API call discovers entry points; it must not decide account ownership. This runnable Python check reads the official v1 base URL and key from the environment, sets the HTTP method explicitly, surfaces response bodies on failure, and backs off on `429` while honoring a numeric `Retry-After` value.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def list_oauth_providers(max_attempts=4):
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{base_url}/auth/oauth/providers"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"OAuth provider request failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("OAuth provider request exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(list_oauth_providers(), indent=2, sort_keys=True))
```

The call only establishes available entry points. After Google or GitHub returns an identity, the server resolves that identity before it decides whether to create, sign in, or offer a deliberate link. The database transaction should reject a provider-and-subject pair that is already attached elsewhere. A player asking to unlink Google must first retain email, phone, GitHub, or another verified login route; otherwise the operation fails closed without deleting the identity.

Consider the awkward migration case because it exposes the architecture. Player `u_1842` owns progress and purchases, has an email login, and arrives through GitHub during the cutover. The GitHub identity is resolved first. If it is already bound to `u_1842`, sign-in can continue. If it is unbound, the game asks for recent proof of the existing email login before adding the GitHub identity in the same transaction that checks uniqueness. If it belongs to `u_9917`, the system stops. A coincident email string does not overrule that conflict, and support staff should not “fix” it by moving the identity without a separately authenticated recovery process.

This is where migrations usually become data migrations rather than login-screen projects. Preserve the internal user ID, import explicit identity relationships, rehearse collision handling, and keep an auditable result for every record: linked, held for review, or rejected. Exact batching and rollback mechanics depend on the old provider's export guarantees; your mileage may vary, so test those guarantees instead of inferring them from a successful demo login.

## 4. Name the failure boundaries before cutover

OAuth callback failure must not create a partial link. A uniqueness conflict must not pick a winner. An unlink request must not strand the account. A migration retry must not duplicate an identity. And loss of one entry point must leave a player with a proven recovery path, not a support-only promise.

Short version: fail closed.

These boundaries also shape observability. Count resolution outcomes and uniqueness conflicts without logging tokens or authentication secrets; reconcile imported identity totals against the source export; and sample the held-for-review queue before raising traffic. OWASP's authentication guidance is the baseline for transport, reauthentication, error handling, and session decisions, but the game's own invariants decide whether a technically valid provider identity may control an existing player record.

## 5. Reject automatic matching, but keep its valid use case narrow

The rejected option is automatic account merging based on an email-like or profile attribute. It looks attractive because it reduces prompts during cutover, yet it collapses identity proof and account ownership into one heuristic. That is not suitable for a game where the wrong merge can transfer progress or purchases.

Automatic matching still has a valid, narrow use case outside the ownership decision: it can rank possible records for a support review, provided it never performs the merge and never reveals private account details to the claimant. Stick with a managed provider's native linking flow when it already preserves stable user identifiers, its export and recovery behavior meet the game's requirements, and migration would add risk without changing the boundary. Move only when the new design improves a named invariant or removes an operational constraint you can verify.

The final acceptance test is therefore mundane and demanding: the same player can enter through Google, GitHub, email, or phone and reach the same internal user only after explicit identity resolution; duplicate binding loses the race safely; and removing one method cannot remove the last method. Everything else is interface choice.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6749
- https://auth0.com/docs/authenticate/identity-providers/social-identity-providers
- https://firebase.google.com/docs/auth
- https://docs.aws.amazon.com/cognito/
- https://supabase.com/docs/guides/auth
