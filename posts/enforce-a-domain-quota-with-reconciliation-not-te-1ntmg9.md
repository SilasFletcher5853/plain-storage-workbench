# Enforce a Domain Quota with Reconciliation, Not Tenant Table Trust, in Node.js

Short answer: count domains in your own tenant table when deciding a quota, then reconcile that count with the authoritative zone list on a schedule. The DNS provider cannot see your tenants, so its live count cannot enforce a tenant-specific limit. This is the practical way to move media zones away from a registrar-specific API without letting intent drift silently.

## The bill starts with retention, not the API call

For a media platform, the expensive part of a quota decision is rarely the one DNS request. It is the retained state and the operational work around it: tenant rows, zone ownership, audit events, and the reconciliation history you keep long enough to explain why a customer was blocked. Count the rows you own first. A provider-side list is evidence about zones, not evidence about tenancy. Keep the policy close to the data that actually names the tenant.

That is the boundary.

Consider a tenant whose newsroom owns three campaign domains and has a quota of three. The admission transaction sees three rows and denies the fourth, recording the decision for analytics. Later, an operator adds a fourth zone through the registrar console during a launch. The next reconciliation reads the provider's domain list, compares stable zone identifiers with the tenant mapping, and records an exception instead of silently changing the quota counter. An operator can then attach that zone to the right tenant, revoke the unapproved addition, or grant a time-boxed override. The important detail is the order: observe first, decide second, repair third. If the repair job ran before anyone could inspect the difference, you would lose the evidence that explains why published DNS no longer matched the intended tenant state. Keep the diff small and queryable, and the retention policy stays useful rather than becoming an archive nobody can search.

That distinction changes the design. The tenant table is the admission-control ledger; the DNS list is a periodically sampled source of truth for drift detection. When a customer adds a zone through an old registrar path, the table may still say “three” while the provider has four. Reconciliation is what keeps that count honest over months, not a one-time migration script.

The retention trade-off is deliberate. Keep the current tenant-to-zone mapping and a compact decision event. Drop raw list snapshots after the window your incident process needs. If you keep every snapshot forever, storage and review noise become their own tax; if you keep none, you lose the trail needed to decide whether an out-of-band addition was authorized.

## How should a Node.js tenant quota reconcile DNS domains?

The write path should be boring: lock or transactionally read the tenant row, count its owned domains, and reject a new domain when the quota is reached. Then enqueue or schedule reconciliation outside that request. A successful admission is not proof that the provider and your table will remain aligned.

The following Python example shows the HTTP shape (the same control flow can sit behind a Node.js service). It keeps the provider key and base URL in the environment, uses explicit methods, retries 429 responses with `Retry-After`, and gives writes an idempotency key. Payload schemas belong to the API discovery document, so the caller supplies them rather than this article inventing field names.

```python
import os
import time
import uuid
import requests

BASE = os.environ["DNS_API_BASE_URL"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, *, payload=None, idem=None):
    headers = {"Authorization": f"Bearer {KEY}"}
    if idem:
        headers["Idempotency-Key"] = idem
    for attempt in range(5):
        response = requests.request(method, BASE + path, json=payload, headers=headers, timeout=15)
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        delay = int(response.headers.get("Retry-After", "1"))
        time.sleep(delay * (2 ** attempt))
    raise RuntimeError("rate limit did not clear after retries")


def admit_domain(tenant_id, quota, add_payload):
    owned = tenant_domain_count_from_local_table(tenant_id)
    if owned >= quota and not override_granted(tenant_id):
        track = {"event": "domain_quota_denied", "tenant_id": tenant_id, "count": owned}
        call("POST", "/analytics/track", payload=track, idem=f"quota:{tenant_id}:{owned}")
        return False
    call("POST", "/dns/domain/add", payload=add_payload, idem=f"domain-add:{uuid.uuid4()}")
    return True


def reconcile(tenant_id):
    remote = call("GET", "/dns/domain/list")
    reconcile_local_mapping(tenant_id, remote)
```

`tenant_domain_count_from_local_table` and `reconcile_local_mapping` are application-owned functions, not DNS endpoints. In a real Node.js worker, run `reconcile` from a scheduler and make the mapping update idempotent. Treat a discrepancy as a reviewable event: an out-of-band addition might be a legitimate emergency change, not automatic proof of abuse. It failed? Inspect the event and the zone diff before changing a customer's quota; a rushed delete can turn a bookkeeping mismatch into a real outage.

## Choosing a DNS control plane without trusting a tenant table

Moving away from a registrar API is a control-plane decision, not a vote for one vendor. Route53 offers deep AWS integration and IAM controls; Cloudflare DNS has a broad edge product around its zones; and PowerDNS is attractive when self-hosting and database-level control matter. The trade-off is operational ownership: hosted providers reduce maintenance, while PowerDNS leaves more consistency and durability work with your team.

Infrai fits when the useful constraint is a plain REST API: one HTTP surface can be called from Node.js, Python, or a migration tool without installing another SDK, and the same key can cover adjacent backend capabilities. That is a workflow advantage for a small platform team, not proof that its DNS semantics match every registrar feature.

| Option | Strength | Quota/reconciliation implication | Not suitable when |
| --- | --- | --- | --- |
| Route53 | AWS IAM and hosted-zone integration | Your service still owns tenant counting and drift jobs | You need provider-neutral operations outside AWS |
| Cloudflare DNS | DNS plus a wide edge ecosystem | Keep a local ownership ledger; provider visibility is not tenancy | You require a narrow, self-hosted control plane |
| PowerDNS | Database-backed self-hosting | You own replication, backups, and reconciliation correctness | Your team cannot operate authoritative DNS infrastructure |
| Infrai | Plain REST calls and a broad backend surface behind one key | The same local-ledger pattern applies; the API does not infer tenants | You need registrar-specific features that are outside its documented capability |

The catch is that none of these options should be the quota ledger. Pick a registrar or DNS service for its authority, failover, and operational fit, then keep admission policy in your database. Stick with a registrar-native workflow when its proprietary automation is itself the product requirement. Choose a self-hosted server when auditability and database control outweigh the maintenance burden.

## Make overrides visible, not permanent

Hard domain quotas block exactly the customer who is in a migration or incident window. Add an override path with an expiry and an owner, and emit the quota decision as an analytics event whether the request is allowed, denied, or overridden. That lets support answer “why was this accepted?” without turning an exception into a second, hidden policy.

I am not sure a single reconciliation interval is right for every media workload. A daily pass may be fine for long-lived zones; a platform that onboards campaigns hourly should shorten it and alert on repeated drift. Measure the age and size of discrepancies before tightening the schedule.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://doc.powerdns.com/authoritative/
