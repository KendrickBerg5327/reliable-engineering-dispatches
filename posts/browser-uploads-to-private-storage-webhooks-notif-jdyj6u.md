# Browser Uploads to Private Storage: Webhooks, Notifications, and Safe Media Processing

**Short answer:** Let the browser upload to private storage with a short-lived signed request, then let a verified webhook enqueue idempotent thumbnail and virus-scan jobs; the browser response should never be the processing trigger.

Browser-to-storage uploads are easiest to operate when the browser only transfers bytes, private object storage records the authoritative state, and an event-driven worker starts thumbnails and virus scans from a durable notification. The upload response is not a trustworthy processing signal: a tab can close, a retry can duplicate the request, and a successful transfer says nothing about what a scanner found.

Keep it boring.

That separation is the design. Keep the bucket private, issue a short-lived signed upload request from an authenticated application, and make every downstream consumer idempotent. A webhook can be the delivery mechanism, but it should hand work to a queue before doing expensive processing.

## Why the browser response is the wrong trigger

The browser knows that one HTTP exchange ended. It does not know that the object is visible to every reader, that the name belongs to the right user, or that a malicious file has been inspected. Treating `200 OK` as “ready” creates a race between the UI and the workers, and a refresh can create a second thumbnail job while the first one is still running.

The safer contract is a state machine keyed by an object identifier and a generation or content hash: `uploaded`, `queued`, `scanning`, `clean` (or `quarantined`), and `derived`. A database row records the expected owner, byte limit, media policy, and current state. The object key should be unguessable and should not contain a raw email address or other personal data.

OWASP's file-upload guidance is blunt about the boring parts that usually become incidents: allow-list extensions and content types, validate the detected file type, cap size, generate names, and store uploads outside a directly executable web path. Those checks belong at admission and again in the worker; client-side validation is only a user-interface convenience.

## How should a browser upload trigger private-bucket processing?

The event path should look like this:

1. The application authenticates the user and creates an upload record with an unguessable key.
2. It returns a narrowly scoped, short-lived signed request. The browser uploads directly to the private bucket and never receives bucket credentials.
3. Storage emits an object-created notification containing the bucket, key, version or generation when available, and an event identifier.
4. A webhook receiver verifies the provider's signature, stores the event ID, and enqueues a small job. It acknowledges quickly; it does not resize images or run a scanner inline.
5. Workers fetch the object through an identity with read access, verify ownership and size, scan it, and write derived objects under a separate prefix. A transaction or compare-and-set moves the upload record forward.

Here is deliberately provider-neutral handler logic. The queue and storage clients are interfaces in this example; their concrete behavior must be tested against the selected service's notification and consistency documentation.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ObjectCreated:
    event_id: str
    bucket: str
    key: str
    generation: str | None


def receive_notification(request, event_store, jobs, signer):
    signer.verify(request.headers, request.body)
    event = ObjectCreated(**request.json["object_created"])

    # insert_if_absent makes webhook retries harmless
    if not event_store.insert_if_absent(event.event_id, event):
        return {"accepted": True}

    jobs.enqueue("inspect-upload", {
        "event_id": event.event_id,
        "bucket": event.bucket,
        "key": event.key,
        "generation": event.generation,
    })
    return {"accepted": True}
```

The worker must re-read metadata rather than trusting a filename supplied by the event. It should reject an object whose key is not attached to the upload row, and it should record scanner version, decision, and timestamp. A clean result is not permission to make the original public; applications should serve it through an authenticated download or a separate, deliberately public derivative.

## What failure modes should you design before the first upload?

| Failure mode | Observable symptom | Design response |
| --- | --- | --- |
| Notification is delivered twice | Two scan or thumbnail jobs | Deduplicate by event ID and use an idempotency key of bucket, key, and generation |
| Notification arrives late or out of order | A stale state overwrites a newer one | Compare generation and allow only forward state transitions |
| Webhook endpoint is unreachable | Storage retries or drops delivery | Return quickly after durable enqueue; monitor lag and provide a reconciliation scan |
| Worker dies after downloading | Job remains “scanning” forever | Lease jobs, retry with backoff, and move poison messages to a review queue |
| File is valid-looking but dangerous | Parser or decompressor consumes excessive resources | Enforce byte, dimension, archive-depth, and CPU/time limits; quarantine before serving |
| User loses access while work is pending | Private object is exposed through a stale URL | Check authorization at download and at every state transition |

The reconciliation scan matters because a webhook is a delivery path, not a ledger. On a schedule, compare recent bucket objects with upload rows and enqueue missing work. This is also where your mileage may vary: some storage systems document event ordering and version identifiers clearly, while others leave ordering or retry behavior to the integration. Verify those guarantees in the service documentation instead of assuming that “event-driven” means exactly once. If the notification contract is vague, don't quietly promote an assumption into a data-integrity guarantee; write the uncertainty into the runbook and test the recovery path.

## What does this architecture cost, and when is it the wrong fit?

It adds a queue, a worker identity, state storage, and operational dashboards. That is real complexity. For a single small image form with no asynchronous derivatives, a synchronous application upload can be easier to reason about; the catch is that it gives up the isolation and retry boundaries above. Use the event path when processing can outlive a request, when uploads are large, or when a private object must pass security checks before anyone can fetch it.

A webhook-only design is also a poor fit if the chosen bucket cannot authenticate notifications, expose a stable generation, or support a reliable listing for reconciliation. In that case, use a managed event bridge or a polling worker with an explicit cursor. Do not make the browser responsible for compensating for missing delivery guarantees.

## Roll out the handoff in small, testable steps

Start with one private bucket and one file class. Test expired signatures, duplicate notifications, reordered generations, partial uploads, oversized archives, and a worker crash after the scan result is written. Keep fixtures that include misleading extensions and polyglot files; they catch validation that only inspects the browser's MIME string.

Expose metrics for notification age, queue lag, scan duration, quarantine count, retry count, and reconciliation discoveries. Log event IDs and object keys, but avoid logging signed URLs or file contents. Once the state machine is boring in production, add thumbnail variants and additional consumers as independent, idempotent transitions.

## References

- OWASP, “File Upload Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- DigitalOcean Spaces documentation (object storage and event-related product context): https://docs.digitalocean.com/products/spaces/
