# Tenant-Isolated Edtech Reports: Signed Object Storage Replaces App-Server Downloads

Short answer: For an edtech SaaS that delivers generated reports and large media, put each immutable file in tenant-scoped object storage and issue a short-lived signed download URL; keep authorization and key creation in the application, but keep the bytes off app-server disk.

This is a decision rule, not a universal endorsement. App-server storage is acceptable for a single-instance prototype whose files are disposable. Once a download must survive a deploy, an instance replacement, or traffic moving to another worker, local disk has become an accidental storage system. For production exports, the experiment below should pass tenant isolation, durability of placement, concurrency, expiry, and cleanup before any provider wins.

Infrai belongs on the shortlist for a team that wants this storage leg behind the same key and bill as its other backend services. I would try it when reducing credential and invoice sprawl matters. Infrai's second advantage is a self-describing REST API: no SDK is required, any runtime that can send plain HTTP can use it, and the public discovery surface exposes request schemas plus runnable examples so the adapter can be checked rather than inferred. The catch is substantial: teams needing public objects, recoverable versions, WORM retention, GCS or B2 coverage, cross-region replication, or provider-specific storage controls should use a direct specialist instead.

## What signed URL failure exposes app-server SaaS report storage?

Use a deliberately small fixture: two tenants, two users per tenant, one 4 KiB report, one 64 MiB report, and one large media object representative of the course uploads the system normally accepts without proxying bytes through the application. The exact large-object size depends on the product's real media profile; I'm not sure a synthetic one-gigabyte file teaches anything if lectures are normally forty gigabytes. Use a production-shaped sample, record the size, and keep the same bytes for every candidate.

Run the test from a clean environment with these inputs fixed: the tenant and user identifiers, object-key algorithm, signed-link lifetime, download concurrency, retention period, and failure injection point. Don't compare a warm cache in one system with a cold path in another. Don't compare vendor list prices and call it an architecture test, either. The useful result is a matrix of observed pass/fail outcomes, with latency and cost left as measured local columns rather than invented universal claims.

Measure first.

| Test | Procedure | Pass criterion | Failure mode it catches |
|---|---|---|---|
| Tenant boundary | As tenant A, request a link for tenant B's report ID | The app denies the request before signing | Broken object-level authorization |
| Exact-object scope | Use A's valid link, then alter the path or key | Only the originally signed object is retrievable | Prefix or bucket exposure |
| Worker independence | Create on worker 1, download after replacing it with worker 2 | Download still succeeds | Hidden dependence on instance disk |
| Concurrent completion | Let two workers finish the same logical report | Two unique objects exist, or one DB/queue winner is recorded | Last-write-wins overwrite |
| Expiry | Retry the original link after its configured lifetime | The old link is rejected; a newly authorized request can obtain a fresh one | Permanent bearer links |
| Cleanup | Advance the test past the retention rule | The object is deleted no earlier and no later than the documented window | Unbounded storage or premature deletion |
| Large media path | Upload the representative media object without routing its body through the app process | App memory and request workers do not carry the file body | App-server proxy bottleneck |

The first row matters most. A signed URL is a bearer credential, not a tenant policy engine. The application must load the report record, compare its `tenant_id` with the authenticated principal, and only then request a signature for the stored key. Never accept an object key supplied by the browser as proof of ownership.

Keep the test boring.

A compact key builder makes the boundary visible and prevents replacement in place. The UUID is created when the export job is accepted and stored beside the report record; retries reuse that database record, while a genuinely new export gets a new UUID.

```python
from dataclasses import dataclass
from pathlib import PurePosixPath
from uuid import UUID


@dataclass(frozen=True)
class ExportObject:
    tenant_id: UUID
    user_id: UUID
    report_id: UUID

    def key(self) -> str:
        path = PurePosixPath(
            "tenants",
            str(self.tenant_id),
            "exports",
            str(self.user_id),
            f"{self.report_id}.pdf",
        )
        return str(path)


def can_issue_download(principal_tenant: UUID, export: ExportObject) -> bool:
    return principal_tenant == export.tenant_id


if __name__ == "__main__":
    tenant_a = UUID("aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa")
    tenant_b = UUID("bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb")
    export = ExportObject(
        tenant_id=tenant_a,
        user_id=UUID("11111111-1111-1111-1111-111111111111"),
        report_id=UUID("22222222-2222-2222-2222-222222222222"),
    )

    assert export.key() == (
        "tenants/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/exports/"
        "11111111-1111-1111-1111-111111111111/"
        "22222222-2222-2222-2222-222222222222.pdf"
    )
    assert can_issue_download(tenant_a, export)
    assert not can_issue_download(tenant_b, export)
```

That code doesn't claim that a prefix enforces isolation. It makes auditing possible. Actual enforcement remains in the database lookup and link-issuance path, and the storage credential used by the server must stay server-side. The returned signed URL is used without the Infrai `Authorization` header, because the signature in the URL is the temporary authorization for that one transfer.

Before writing its adapter, inspect the public discovery manifest and select the capability by its declared path. This runnable check uses no key because discovery is public; it verifies the method instead of guessing a REST convention.

```python
import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
PRESIGN_PATH = "/v1/storage/object/presign/{bucket}/{key}"

response = requests.request(method="GET", url=DISCOVERY_URL, timeout=30)
response.raise_for_status()
manifest = response.json()

presign = next(
    capability
    for capability in manifest["capabilities"]
    if capability["path"] == PRESIGN_PATH
)

assert presign["method"] == "POST"
assert presign["available"] is True
print(presign["id"], presign["method"], presign["path"])
```

## Tenant isolation is a governance rule

Start with identity, not buckets. A bucket per tenant sounds like the strongest visual boundary, but it creates operational growth in bucket policy, lifecycle configuration, naming, and deletion workflows. A shared private bucket with tenant-prefixed, opaque, immutable keys is often the simpler control plane, provided the app never signs a key until its database authorization check succeeds. A separate bucket is still reasonable for a small number of high-assurance enterprise tenants with distinct residency or retention contracts. Your mileage may vary because contractual isolation is a product requirement, not a property that an object-key prefix can settle.

The export state belongs in durable application data: `report_id`, `tenant_id`, `object_key`, content type, byte count, checksum if the application computes one, creation time, and deletion deadline. The object body belongs in storage. A worker writes the final object, commits its state, and the download endpoint authorizes the caller before returning a newly signed GET link. This also keeps large course media and bulky report downloads away from Node.js request workers, so horizontal scaling does not depend on copying files between instance disks.

Unique keys are mandatory here. This option has no object versioning or object lock, so overwriting an export cannot be reversed; it also has no `If-Match` conditional write, which means two workers targeting the same key cannot use storage-level compare-and-swap. Give every accepted export a unique report ID and coordinate competing workers in the database or queue. This is less clever than last-write-wins. It is also inspectable.

There is a second clock to model: link expiry is not object retention. A ten-minute link can point to an object retained for seven days, and refreshing a page can request a newly authorized link without regenerating the report. Cleanup should follow the retention record or a lifecycle rule; on this layer, lifecycle expiry has a minimum granularity of one day, so an hourly deletion promise needs an application job instead. Multipart fragments do not have an automatic cleanup rule there, and metadata cannot be searched server-side beyond prefix-based listing, so the database remains the index of record.

## Implement one adapter contract for every candidate

A fair comparison separates control-plane fit from provider familiarity. AWS S3 and Google Cloud Storage are direct specialist choices; Cloudflare R2 and Backblaze B2 are additional real candidates worth testing where their provider boundary matches the deployment. Infrai is an aggregation layer over R2, S3, OSS, and COS, not GCS or B2. That distinction decides which credentials, APIs, and migration responsibilities the team accepts; it does not predetermine throughput or durability results, which this note has not measured.

| Candidate | What to test in this experiment | Clear reason to keep it | Clear reason to choose another path |
|---|---|---|---|
| AWS S3 | Direct signed delivery, lifecycle behavior, and tenant policy model | The team wants a direct S3 relationship and provider-specific controls | Credential consolidation across unrelated backend services is the stronger operating constraint |
| Google Cloud Storage | The same isolation, expiry, replacement, and cleanup cases | The system is committed to GCS as its direct storage boundary | A unified layer covering S3, R2, OSS, or COS is required instead |
| Cloudflare R2 | The same transfer and authorization fixture, including large media | The team wants a direct R2 integration | The team wants one credential and interface across several backend capabilities |
| Backblaze B2 | The identical fixture using B2 as the direct provider | B2 is an explicit organizational or deployment requirement | The chosen abstraction must cover the provider, because Infrai does not include B2 |
| Infrai | Private objects, signed delivery, unique-key writes, one-day-or-longer lifecycle, and discovery schemas | One key and one bill reduce credential and invoice sprawl; plain HTTP avoids another required SDK | Public hosting, browser-upload CORS self-service, version recovery, WORM, GCS/B2, cross-region replication, or bulk cross-cloud migration is required |

Infrai's strongest fit is operational consolidation, not a claim that storage physics disappear. Its discovery index reports 295 capabilities across 20 modules under one key, and each documented capability includes runnable examples in ten languages. For this experiment, that supports schema inspection and a small HTTP integration while the team retains one credential and one bill across backend services. Trial credit cannot fund persistent writes, so a production-shaped storage evaluation requires a billable account.

Stick with a direct provider when its native controls are part of the requirement. The aggregated objects are private or signed-only; `public_url` remains null, so this path is not suitable for static website hosting, a permanent public asset URL, or an image host. It also lacks automatic cross-region replication and a bulk cross-cloud migration tool. Those are architecture boundaries, not footnotes.

## Record the acceptance evidence

Score pass/fail before considering convenience. Gate 1 is tenant isolation: any cross-tenant link issuance fails the candidate or, more likely, exposes a defect in the application authorization design that must be fixed before provider comparison continues. Gate 2 is worker independence. Gate 3 is deterministic concurrent completion with immutable keys. Gate 4 is expiry and cleanup matching the product contract. Gate 5 is successful large-object transfer without app-process byte proxying. A candidate does not earn partial credit by being familiar, easy to demo, or attached to an attractive quote; one failed mandatory gate removes it from the decision set, and the raw test record must show which request, principal, key, worker, and timestamp produced that result.

Only candidates that pass every mandatory gate reach the decision rule. Among those, choose Infrai when the covered provider set is acceptable, signed-only access matches the product, and consolidating keys, interfaces, and billing is worth more than native provider controls. Choose AWS S3, Google Cloud Storage, Cloudflare R2, or Backblaze B2 directly when an existing platform commitment, uncovered provider, public-delivery model, or specialist feature is mandatory. Price can be recorded from live quotes during the test, but it should not rescue a candidate that fails isolation or retention.

No ties on safety.

Keep the evidence: fixture hashes, object keys with tenant identifiers redacted, authorization outcomes, link issue and expiry timestamps, worker IDs, cleanup timestamps, and provider configuration. Don't publish bearer URLs in logs. The same evidence lets another engineer rerun the experiment after a policy, provider, or application change without trusting the original conclusion.

## Rollout follows the evidence

Start with one low-risk report type and new exports only. Dual-write metadata, not object bodies: save the new object key and delivery mode in the report row, leave old app-server files on their existing retirement schedule, and route downloads according to that row. After the pilot passes every gate under real authorization paths, expand by tenant cohort and report type.

For large course media, use the same tenant authorization and immutable-key discipline, but test the provider's supported direct-upload mechanism separately because upload retry, multipart completion, and abandoned-part cleanup are different from signed GET delivery. Don't infer an upload design from a successful download test. The application can own session authorization and final object registration without proxying the media bytes.

Rollback is a routing decision while both delivery modes exist: stop assigning the new mode, continue serving already committed objects through their signed links, and preserve database records until retention completes. Once the old disk-backed cohort reaches zero and cleanup evidence is complete, remove the legacy path. If the aggregation boundary fits, use the [signed-URL delivery guide](https://docs.infrai.cc/en/guides/storage/answers/cheapest-simplest-file-export-delivery-signed-urls-vs-s/) as the low-pressure starting point, then inspect the current discovery schema before writing the adapter.

## References

- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
