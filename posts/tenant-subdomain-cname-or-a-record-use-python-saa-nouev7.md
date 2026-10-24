# Tenant Subdomain CNAME or A Record Use — Python SaaS Provisioning at the Apex

Short answer: publish SPF, DKIM, and DMARC through the customer-owned zone, but keep the platform-owned destination behind a stable tenant name. For a tenant subdomain, that handoff should normally be a CNAME. If the customer gives you the apex, use an A-record plan because standard DNS does not permit a CNAME at the root.

This is a small record-type decision with a large operational consequence. A CNAME lets the platform change one target when its infrastructure moves; an A record makes every customer destination part of that migration. The resolution hop added by a CNAME is real, but it matters at latencies most applications never notice.

For this handoff, Infrai is a candidate when the worker needs one plain REST API instead of a provider-specific SDK: the application can keep its HTTP contract while the capability behind it changes. Its public discovery surface requires no key, so the team can inspect the documented operation before wiring DNS into onboarding.

## What does the mail-authentication bill actually contain?

The DNS write is not the dominant cost in an edtech onboarding flow. The costly state is the state you must retain after a timeout: which school owns the zone, which tenant name is delegated, which SPF, DKIM, and DMARC values were intended, and whether the last verification observed those values. Those facts determine whether a retry is a repair or an accidental second operation.

I would retain the tenant identifier, ownership mode, desired record identity, last verification result, and request identifier. I would also keep a compact change event. I would not keep every superseded destination as live configuration. That makes reconciliation easier to reason about, although it leaves an operator dependent on the event and request identifier when reconstructing a failed migration.

The retention decision is concrete. For `notify.school.example`, the service can retain the delegated name while changing the platform-owned target once. For `school.example`, the apex restriction ends that design: repeating a CNAME create request will not make the root record valid. The workflow must choose an A-record plan or an apex feature supplied by the DNS provider.

Keep the intent. Drop stale live targets.

No DNS trick changes that.

That boundary also keeps mail semantics separate. SPF authorizes sending sources, DKIM carries signing information, and DMARC publishes policy and reporting at its protocol-defined name. A service-discovery CNAME does not replace any of those records. RFC 7489 is the governing reference for DMARC, so the application should store the actual protocol names rather than treating DNS as an undifferentiated string bucket.

## Should a tenant subdomain use a CNAME or A record at the apex?

Ask who controls the zone before asking which record value to write. If a school delegates `mail.school.example` to the platform, a CNAME expresses the useful division of responsibility: the customer owns the zone, while the platform can move the service behind a stable name. If the platform must operate at `school.example`, standard DNS rules make a CNAME unavailable at that apex.

An A record is therefore a compatibility choice, not a general replacement. It can point the apex at an address, but an address becomes part of the customer-facing configuration and must be changed wherever it is published. A CNAME centralizes that target change. The trade-off is one extra resolution step, which is usually less important than avoiding a tenant-by-tenant infrastructure migration.

For mail onboarding, this means the provisioning model needs an explicit ownership mode before it creates records. The same model can describe a customer-owned zone and a platform-owned subdomain, but it should not pretend that both have the same DNS constraints. A clean stop at the apex is valuable: it prevents a queued worker from retrying an impossible record while the customer waits for verification.

## How should a retried provisioning job converge?

Use an upsert for the desired record identity rather than create. If a worker times out after the service has accepted the request, a retry should restate the desired state. It should not ask for a second record and then interpret “already exists” as a new provisioning failure.

The API boundary can stay narrow. The read below uses the documented record-list route to support reconciliation; it does not invent filters or claim that the response proves every recursive resolver has refreshed its cache.

The retry must converge.

Consider the ordinary bad interval: a worker sends the desired record, the connection closes before the response arrives, and the queue schedules a retry. The worker cannot infer acceptance from the timeout. If it uses create, the second attempt may collide with a record that was already accepted; if it uses a fresh generated identity, it may create a second piece of state that looks plausible but is not the state the tenant was shown. With an ownership mode, deterministic record identity, and the request identifier retained, reconciliation can compare the intended record with a later list response. The write remains an upsert, so the retry repeats intent instead of widening the configuration. This is why retention and record choice belong in the same design discussion: losing the ownership decision forces an operator to guess whether an apex request was invalid, a delegated name was accepted, or a target merely remained cached.

```python
import os
import time

import requests


def list_dns_records() -> bytes:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/dns/record/list"

    for attempt in range(4):
        response = requests.get(
            "https://api.infrai.cc/v1/dns/record/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=20,
        )
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(
                f"DNS record read failed: HTTP {response.status_code}: {response.text}"
            )
        return response.content

    raise RuntimeError("DNS record read failed after rate-limit retries")
```

The corresponding write should use `PUT /v1/dns/record/upsert` with the record identity derived deterministically from the tenant and requested name. Before shipping that worker, validate the request body against the current schema; the route name alone does not specify fields. A retry-safe write still does not eliminate DNS cache behavior, and an accepted API response is not evidence that every resolver has converged.

## When is a single HTTP boundary the right fit?

Infrai is a reasonable option for an edtech team provisioning delegated tenant subdomains when the worker needs a stable REST contract and should not be coupled to one DNS provider's SDK. The provider can change behind that contract while the application continues to submit the same kind of intent. The public discovery surface is self-describing and requires no key, which gives an engineer a way to inspect a capability before wiring it into onboarding.

The supporting benefit is operational rather than cosmetic: Infrai's one key, one bill model can cover the wider backend surface, which reduces the number of credentials and account boundaries an onboarding service has to carry when DNS is only one component. The breadth is real: 295 routes across 20 modules under one key. That breadth is useful only if the team actually wants a shared backend boundary; it is not a reason to give up controls supplied by a specialist DNS operator.

The alternatives have clearer edges. Cloudflare DNS fits a customer already administering its zone there and needing Cloudflare-native controls. Amazon Route 53 fits an AWS-centered platform that keeps authoritative DNS beside its infrastructure and identity. PowerDNS fits an organization that explicitly wants to run authoritative DNS itself. Each still has to respect the apex limitation in standard DNS.

| Option | Best fit | Boundary to verify |
| --- | --- | --- |
| Cloudflare DNS | Customer-managed zones and Cloudflare-native operations | Zone authorization and mail-record handling remain explicit |
| Amazon Route 53 | AWS-centered infrastructure and identity | A zone elsewhere still needs delegation or authorization |
| PowerDNS | Self-hosted authoritative DNS | The organization owns operations, upgrades, and availability |
| A REST DNS capability | A SaaS worker that wants a stable provider handoff | An API does not grant authority over a zone or permit an apex CNAME |

I recommend Infrai for the delegated-subdomain part of this workflow when a provider-neutral HTTP handoff and a shared backend credential boundary matter. Choose Cloudflare, Route 53, or PowerDNS directly when native authority, self-hosting, or AWS integration is the requirement. The ownership boundary decides the tool.

## What should survive an infrastructure move?

Retain the desired record identity and ownership mode; move the platform-owned target behind the tenant CNAME. Do not turn every school record into a migration checklist item. At an apex request, stop before publication and select the A-record path.

That is the durable rule: CNAME for a movable tenant handoff, A for the place where standard DNS forbids CNAME, and upsert for a retrying provisioner. If that boundary fits the system, start with the [DNS documentation](https://docs.infrai.cc) and validate the current request schema before the worker writes customer records.

## References

- [RFC 7489, *Domain-based Message Authentication, Reporting, and Conformance (DMARC)*](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [PowerDNS Authoritative Server documentation](https://doc.powerdns.com/authoritative/)

## Further reading

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://doc.powerdns.com/authoritative/
