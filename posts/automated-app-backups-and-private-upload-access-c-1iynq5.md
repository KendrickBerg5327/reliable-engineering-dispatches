# Automated App Backups and Private Upload Access: Compatible Storage Lifecycle Rules

Short answer: keep user-uploaded objects private, let the application authorize each download, and use short-lived signed URLs only after that authorization. S3-compatible storage with lifecycle rules is a reasonable implementation for private uploads and daily app backups, but the lifecycle policy should clean up data; it should not decide who may download it.

That separation is the decision. Delivery is simple when a browser receives a signed URL, while access control stays simple when the application remains the policy engine. Mixing those responsibilities creates a quiet failure: an object expires on the intended schedule, yet an already-issued link still grants access longer than the product owner expected. The link is temporary authority, not a privacy setting.

## Governance: private upload access and daily backup retention

Write the invariants before comparing storage services. Every object has a tenant-scoped key that the client cannot choose freely. Every download begins with an application authorization check. The storage layer sees a private object request, not a tenant role. Lifecycle rules remove unreferenced or expired material after the product's retention decision has been recorded. A successful upload does not by itself make an upload visible in the application.

For a B2B SaaS application, a useful state model is `received`, `verified`, `published`, and `expired`. The database owns that state. The object store owns bytes and their deletion schedule. A background job can reconcile records whose object is missing, whose upload never completed, or whose retention date has passed. This gives support engineers an answer better than “the bucket says it exists.”

Daily app backups belong beside this flow, but they should not be confused with user-upload retention. A 30-day recovery prefix and a 90-day recovery prefix can be useful operational labels; they are not an authorization model. Keep backup keys separate from tenant upload keys, and make the US/EU placement decision explicit for each data class. A retention rule that is correct for a database dump may be entirely wrong for a customer document.

The first test is deliberately boring: create a disposable object, authorize one tenant, request a signed download, revoke the application record, and check that a new signing request is denied. Then test the expiry boundary and the lifecycle deletion boundary independently. They answer different questions.

Keep it private.

Test it twice.

Then revoke it.

The test data should include a published document, an unverified upload, a key from another tenant, and a backup prefix with a 30-day or 90-day retention label. For each case, record the application decision, the storage request, and the resulting audit event. This takes longer than checking one upload acknowledgment, but it exposes the failure that matters: a valid object is not necessarily an authorized object, and a deleted object is not necessarily evidence that the intended retention policy ran. It doesn't matter how familiar the API looks if the acceptance test cannot distinguish those states.

The awkward case is a user who is authorized at upload time and removed from the project while a link is still circulating. The object may remain within its retention window, and the storage system may quite correctly continue to hold it, but a new application authorization must fail. If the product promises immediate revocation of an already-issued URL, a normal signed-link design is the wrong primitive; route the download through a service that can recheck policy for every request. That is a governance choice with a delivery cost, not a lifecycle setting.

## What does the Python implementation need to preserve at the delivery boundary?

The proposed architecture has one narrow critical path. The application authenticates the caller, checks tenant ownership and object state, then asks storage for a signed read constrained to the intended object and lifetime. The browser follows that URL without receiving the application's bearer credential. AWS documents this presigned-URL pattern and its expiry behavior; a signed URL should be treated like a password with a deadline, because anyone who obtains it can use it until that deadline or an earlier storage-side revocation takes effect.

Here is the control flow in Python. `Storage` is an adapter boundary, not a claim about a particular provider's SDK. The important detail is that the object key comes from the database record, never from an unchecked path supplied by the browser.

```python
from dataclasses import dataclass
from datetime import timedelta
from typing import Protocol


@dataclass(frozen=True)
class Upload:
    tenant_id: str
    object_key: str
    state: str


class Storage(Protocol):
    def create_signed_download(
        self, object_key: str, expires_in: timedelta
    ) -> str:
        """Return a URL for this private object only."""


def issue_download_url(
    caller_tenant: str,
    upload: Upload,
    storage: Storage,
) -> str:
    if upload.tenant_id != caller_tenant:
        raise PermissionError("tenant does not own this upload")
    if upload.state != "published":
        raise PermissionError("upload is not available")

    # Keep this lifetime short enough for the product's download workflow.
    return storage.create_signed_download(
        object_key=upload.object_key,
        expires_in=timedelta(minutes=10),
    )
```

The 10-minute value is an example policy, not a universal answer. A support export, a large media file, and a one-click document preview have different delivery constraints. I would record the chosen lifetime as an acceptance criterion and test it with a clock you control, because a UI label saying “download expires soon” is not evidence that the storage request has the same deadline.

The failure boundary is just as important. A `403` from the application means authorization failed; a signed URL that is leaked is a credential exposure; a missing object is an integrity or retention event; and a lifecycle deletion is an expected data-management event only when the record says deletion is allowed. Log the event type, tenant-safe object identifier, policy version, and correlation ID. Do not log the full signed URL.

## How should I compare compatible storage for daily app backups?

“S3-compatible” describes an interface target, not an approval decision. Run the same harness against each candidate: private-by-default creation, upload completion, tenant authorization, signed download, revocation behavior, lifecycle expiry, and a restore of a representative backup. Check the required US and EU locations separately. The object API can look familiar while retention, regional placement, request accounting, or operational tooling differs.

| Option | What the team can verify | Boundary that remains yours | Appropriate when |
|---|---|---|---|
| S3-compatible service | Object operations, private reads, lifecycle configuration, and regional availability through its current documentation | The application authorization model, restore test, and current total request and retrieval terms | The client already targets the S3 protocol and the service passes the harness |
| Direct application proxy | Authorization and audit stay on the application path | The application now carries download bytes, buffering, range requests, and delivery load | Every byte needs inspection or policy enforcement before release |
| Signed direct download | Storage carries the bytes while the application issues temporary authority | Link leakage, expiry semantics, and revocation expectations need explicit tests | The object is already verified and the browser can follow a short-lived URL |
| Archive or backup system | Retention and restore workflows may be purpose-built for recovery data | It may not fit interactive private downloads or tenant-level authorization | The primary job is recovery, immutability, or long retention rather than delivery |

Cost belongs in the worksheet after those tests. Count retained bytes, daily writes, listing and policy operations, and one realistic recovery. A public pricing page, such as the Backblaze B2 pricing page, can supply current inputs for one candidate, but it cannot establish that the candidate's lifecycle behavior or regional terms fit this application. Your mileage may vary when the workload has many small objects or unusually frequent restores.

## Which failure modes make this private delivery design unsuitable?

The catch is that lifecycle automation is deletion automation. It is not WORM retention, object version recovery, malware scanning, legal hold, or cross-region disaster recovery. This design is not suitable when a customer contract requires immutable records, when a revoked user must invalidate every previously issued link immediately, or when regulation requires a retention decision that cannot be represented by the storage service's lifecycle model. Use an archive system or an application-mediated download path for those boundaries, and document the handoff.

It is also the wrong choice for a public asset host. Do not turn a private upload bucket into a static website because the frontend wants a convenient URL. Use a separate public delivery system with a separate data classification. The access model should be visible in deployment review, not inferred from a filename prefix.

The rejected option is “let the bucket lifecycle rule solve authorization.” It is attractive because it removes a database check, but it cannot express tenant ownership, publication state, or a user being removed from a project. Keep it only for disposable staging data whose loss is intentional. For production uploads, the application decides access; storage decides whether the bytes remain.

The durable rule is to separate access from retention: use compatible object storage only after it passes a real private-download and restore test, keep 30- and 90-day backup retention separate from user access, and choose a proxy or archive system when the missing control is more important than delivery simplicity.

## Further reading

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://www.backblaze.com/cloud-storage/pricing
