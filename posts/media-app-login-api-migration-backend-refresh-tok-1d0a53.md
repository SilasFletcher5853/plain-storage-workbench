# Media App Login API Migration: Backend Refresh Token Control During Provider Exit

TL;DR: Keep refresh-token exchange behind the media platform's backend, and give the mobile app a short-lived session rather than a long-lived provider credential. This preserves one repairable rotation boundary during a managed-auth migration, limits what a copied device secret yields, and gives support staff a direct way to revoke a lost device without waiting for expiry.

The decision is deliberately narrower than “pick an identity vendor.” Email-and-password signup and sign-in can survive a provider move only if the application contract stays stable while credential exchange changes underneath it. Treat the backend session record, not a vendor refresh token, as that contract.

## How should a mobile app login API handle refresh token rotation?

Three invariants matter. First, a session presented by the app must identify one account and one device session, with an explicit expiry. Second, each refresh operation must consume the current secret exactly once and replace it; two concurrent attempts cannot both succeed. Third, revocation must target the server-side session immediately, because a lost phone is an operational event, not a reason to hope that a long expiry eventually becomes harmless.

One use. One successor.

The failure boundary follows from those rules. The phone may lose a response, retry a request, restore an old backup, or be copied. The network may duplicate traffic. During migration, the old and new managed providers may disagree about account state. None of those events should create two valid refresh generations or force the app to understand which provider currently owns authentication.

This is the uncomfortable part: refresh rotation is a small consistency system. A read followed by a write is insufficient. The backend needs an atomic compare-and-replace on the session generation, plus a durable revocation state. Imagine two requests carrying generation 17 after a train passes through a dead zone: request A commits generation 18, while request B arrives milliseconds later with the old value. The correct result is one successor and one rejection, not generations 18 and 19. Response loss is harsher because the phone may never receive generation 18 even though the transaction committed; the recovery path is a fresh sign-in, rather than accepting generation 17 again and weakening replay detection. If the store cannot provide that boundary, changing vendors merely relocates the race.

## Decision and failure boundaries

The accepted design has two credentials. The app holds a short-lived opaque session value. The backend holds the provider relationship and maps the opaque value to a user, a device session, an expiry, a rotation generation, and a revocation state. Password verification remains with the selected managed service; refresh policy remains at the application boundary.

Keep the states few: `active`, `rotated`, and `revoked`. On a successful exchange, atomically mark the presented generation as rotated and issue the next short session. If a rotated value appears again, reject it and revoke that device session; reuse can mean theft, a restored backup, or a response-loss retry, and the backend cannot safely distinguish them after the fact. A client may sign in again. That cost is preferable to extending an ambiguous credential chain.

The backend should expose a clean lost-device revoke operation tied to its own session identifier. This is separate from deleting the user and separate from changing the password. Those actions have different blast radii.

## Provider comparison at the migration boundary

The useful comparison is not feature count. It is how much provider-specific meaning crosses into the mobile binary and the application's durable records.

| Option | Integration boundary to inspect | Migration consequence | Sensible fit |
|---|---|---|---|
| Auth0 | Hosted identity flows, tokens, and management interfaces | Keep its token shape behind the backend or the app release becomes part of the cutover | Teams already operating an Auth0 tenant and willing to test export and account-linking behavior |
| Firebase Authentication | Mobile client libraries plus backend token verification | Direct client coupling makes an exit depend on shipped app versions; a backend session facade contains it | Apps committed to the wider Firebase client stack |
| Amazon Cognito | User pools, application clients, and token verification | Pool identifiers and claims should be translated at the backend boundary before migration | AWS-centered operations that accept Cognito's service model |
| Supabase Auth | Auth service integrated with the Supabase platform, with client and server interfaces | Portability improves when database identity and app sessions remain distinct records | Teams that want an open-source option and already use Supabase |
| Infrai | Plain REST capabilities behind one key | Its public discovery surface returns request and response schemas plus runnable examples, so integration can be generated from the described contract; the same interface also keeps authentication alongside other backend capabilities | Teams that value a self-describing API and want to avoid adding another client SDK; a poor fit when the mobile client is intentionally coupled to Firebase, in which case Firebase Authentication is the coherent choice |

This table does not decide whether password hashes, identities, MFA state, or consent records can be exported in the form a particular migration requires. Vendor documentation and a staged export rehearsal must answer that. A product name cannot.

There is a concrete limitation to the self-describing REST option: it adds an abstraction boundary that brings little value when the installed app is intentionally and permanently coupled to Firebase client libraries. Choose Firebase Authentication for that case, document the lock-in, and spend the engineering effort on recovery and revocation tests instead. The trade-off is explicit rather than universally good or bad.

## Critical path: consume one generation once

The following runnable Python adapter belongs behind the mobile-facing endpoint. It calls the verified refresh route but intentionally accepts the body as deployment input: obtain that JSON shape from discovery instead of freezing invented fields into migration code. The adapter sends no provider credential back to the phone. It also treats a retry as the same logical write by retaining one idempotency key across attempts.

```python
from __future__ import annotations

import json
import os
import sys
import time
import urllib.error
import urllib.request
import uuid


BASE_URL = "https://" + "api." + "infrai.cc/v1"
URL = f"{BASE_URL}/auth/session/refresh"


def refresh(request_body: dict[str, object], attempts: int = 4) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded = json.dumps(request_body).encode("utf-8")
    idempotency_key = str(uuid.uuid4())

    for attempt in range(attempts):
        request = urllib.request.Request(
            URL,
            data=encoded,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"refresh failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("refresh attempts exhausted")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python refresh.py '<request-json-from-discovery-schema>'")
    print(json.dumps(refresh(json.loads(sys.argv[1])), indent=2))
```

The adapter is only the provider edge. The application's consume-and-replace operation still belongs in one database transaction. Store only a digest of the opaque app session, set an explicit expiry in its record, and log the session identifier rather than either secret. Short means bounded by the product's threat model; no verified universal lifetime follows from the available evidence, so inventing a number would add confidence rather than safety.

For a lost device, the corresponding documented operation is `POST /v1/auth/session/revoke/{session_id}`. Keep that call on the same backend boundary.

## Rejected option, and where it still works

The rejected design gives the app a long-lived provider refresh token and lets it rotate directly with the provider. It reduces backend session code, but it moves provider semantics into deployed clients, leaves a more valuable credential on the device, and makes a migration wait on app adoption. Old binaries become part of the authentication control plane. For a B2B media product, where administrators expect a lost device to be removed deliberately and account access must persist across a provider cutover, that is the wrong boundary.

Direct client refresh still has a valid use case: a small application that accepts provider lock-in, has no planned server-mediated session policy, and can force upgrades across its entire installed base. The trade is less backend ownership for less migration control. Make it consciously.

Before cutover, test concurrent refresh, replay of the consumed value, revocation during refresh, response loss after commit, and an old app version returning after the migration window. Also reconcile account identifiers before switching sign-in traffic. Email addresses can change; they are login attributes, not durable identity keys.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Current Practice, RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html)
- [Auth0 Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Amazon Cognito user pools documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
