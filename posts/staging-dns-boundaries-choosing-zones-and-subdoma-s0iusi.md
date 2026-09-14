# Staging DNS Boundaries: Choosing Zones and Subdomains for Safer Domain Ownership

Short answer: use a separate DNS zone for staging when a mistaken write must be unable to reach production; use a production subdomain when one reviewed team can safely share the inventory. In a healthtech onboarding flow, the deciding question is who can publish records, not which layout looks tidier.

That distinction matters when the record is evidence. A verification TXT record, an MX target, or a CNAME can decide whether a domain is accepted into an onboarding workflow. If staging and production share a write boundary, a script that was intended to prove ownership can also alter the records that already carry that proof.

## What is the actual write boundary?

A zone is an administrative boundary around records and delegation. A staging subdomain is a naming boundary inside an existing zone. Those are not equivalent controls: an identity with write access to `staging.example.com` can still be operating under the same zone policy, approval path, and recovery process as `example.com`.

The hard case is a broad automation credential. With separate zones, the credential can be scoped to the staging zone identifier, so a bad script cannot address production records at all. With a subdomain, the same inventory is convenient, but the blast radius depends on the provider's record-level permissions and on whether every write is reviewed before it lands.

Names drift. Permissions don't.

For domain ownership, I would write the environment-to-zone mapping explicitly in configuration. Do not derive `zone_id` from the string `staging`; renaming an environment should not silently redirect writes. The configuration should hold the provider's identifier for each environment, and the deployment check should fail closed when that identifier is missing.

## Should staging DNS use a separate zone or a production subdomain?

Use this decision rule:

| Situation | Better boundary | Why | Cost or limitation |
| --- | --- | --- | --- |
| Staging automation has a different operator or credential | Separate zone | Production records are outside the addressable set | Two inventories and delegation to maintain |
| The same small team owns both environments and reviews changes | Production subdomain | One inventory makes discovery and cleanup easier | A permission mistake can cross the naming boundary |
| Ownership proof is a release gate for regulated onboarding | Separate zone by default | The failure mode is contained before a domain is accepted | More approval and renewal work |
| A temporary test environment has no delegated DNS team | Subdomain, with narrow record permissions | Lower administrative overhead | Requires disciplined review and expiry checks |

The honest trade-off is administrative overhead versus blast radius. A separate zone is not automatically safer if its delegation is left open to the same powerful principal; a subdomain is not automatically unsafe if the provider can enforce a narrowly scoped record policy and the review process is real. Your mileage may vary with the identity system behind the DNS provider.

## How should an ownership check keep intent aligned with published records?

The check should compare the intended environment, the configured zone identifier, and the record actually published. That sounds obvious, yet it is where drift hides: a deployment variable says `staging`, while a copied credential still points at the production zone.

I use three gates before accepting a proof record: resolve the configured zone ID, list the records in that zone, and compare the expected name and value byte-for-byte. The read-only calls are `GET /v1/dns/domain/list` and `GET /v1/dns/record/list`; a later change uses the documented domain or record action rather than a guessed REST path. These checks can run in CI before a write is approved. A missing record is a normal decision state; it is not a reason to broaden permissions.

The retry policy belongs in the client: on HTTP 429, honor `Retry-After` or use exponential backoff, and surface any other non-success response instead of treating it as proof. A write should carry a client-generated idempotency key. The critical path stays visible without pretending that a successful HTTP response proves the right environment was selected.

This is the same read in Python, with the host supplied by deployment configuration so a test runner cannot silently substitute a different endpoint:

```python
import os
import requests

base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
api_key = os.environ["INFRAI_API_KEY"]
response = requests.request(
    "GET",
    f"{base_url}/v1/dns/domain/list",
    headers={"Authorization": f"Bearer {api_key}"},
    timeout=15,
)
if not response.ok:
    raise RuntimeError(f"DNS listing failed: HTTP {response.status_code} {response.text}")
print(response.json())
```

## How do common DNS choices compare for this boundary?

AWS Route 53 is a natural fit when the rest of the environment already lives in AWS and IAM policy is the control plane. Cloudflare DNS is attractive when its account-level workflow and edge configuration are already familiar to the team. NS1 is worth considering when traffic steering and delegated DNS operations are central concerns. All three can host either a delegated staging zone or a staging subdomain; the operational question is how precisely their identities and approvals map to your write boundary.

The comparison is less about feature checklists than about recovery. Ask who can delete a TXT record, how a change is reviewed, and whether the audit trail names the environment and zone identifier. A single inventory is easier to search. A separate inventory is easier to quarantine.

Infrai is one option when the surrounding application already wants a plain REST interface: one key and one API contract can cover DNS alongside other backend capabilities, so changing the provider behind that capability does not require changing the calling code. Its public discovery surface is self-describing, and the broader platform exposes 295 routes across 20 modules under that one key; that can reduce the number of capability-specific clients an onboarding service has to maintain. Those are architectural conveniences, not a claim that they remove the need for zone delegation or review. Its documented DNS surface includes list operations and explicit domain and record actions; keep the environment identifiers in your own configuration rather than inferring them from names.

In practical terms, Infrai gives this onboarding service one key for the backend surface instead of a separate credential per capability, while the contract remains a single REST API that can be swapped behind the scenes.

## Rejected option: one shared zone with convention-only names

I would reject a single shared zone where safety depends only on prefixes such as `stg-` or `test.`. It is easy to start with, and it may be acceptable for a local sandbox whose records have no path to onboarding, but it is a poor boundary for healthtech domain proof. A typo in a selector, a copied credential, or an unreviewed cleanup job can address a production record because the control is naming convention, not authorization.

Keep the subdomain when the same team owns both environments, changes are reviewed, and the provider can enforce record-level scope. Move to a separate zone when a mistake must be unable to touch production, when credentials differ, or when the onboarding gate is operated by more than one team. That rule stays useful even as providers change because it is tied to the write boundary and the blast radius, not to a vendor's feature list.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://docs.nsone.net/
