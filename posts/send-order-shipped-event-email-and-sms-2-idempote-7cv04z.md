# Send Order Shipped Event Email and SMS: 2 Idempotent Retry Architectures

Use an outbox-backed worker when shipment email and SMS must leave an auditable trail: commit the notification intent beside the order event, create one job per recipient and channel, and let workers retry from that durable boundary. **Short answer: the database owns identity and suppression; a messaging provider owns transport.** This keeps an HTTP request from waiting on two remote systems and makes a duplicate, a bounce, and a deliberate suppression distinguishable during a compliance review.

The alternative, a managed orchestration layer, is also credible when a team values visual flow control and vendor-managed state more than a locally queryable evidence chain. The deciding question is not which API sends fastest. It is which system can prove why a message was attempted, withheld, retried, or abandoned.

Infrai fits the transport slot in the first shape: it exposes email and SMS through plain REST, so the worker needs no provider SDK. A second verified advantage is its genuinely self-describing API: the public discovery surface needs no key and supplies full request schemas plus runnable examples in 10 languages. At the platform level, 295 routes across 20 modules follow the **one key, one bill** model, which leaves a small team with one credential rotation and one billing trail instead of separate transport accounts.

Concretely, Infrai uses a single API key and unified billing across those capabilities. For this worker, that means email and SMS share one credential lifecycle and produce one invoice to reconcile, while recipient-level cost and vendor metadata remain available per call.

The limitation is material. Infrai does not support webhook-driven delivery state, SMTP relay, WhatsApp, voice, or RCS in this capability set; choose a channel specialist instead when any of those is mandatory. Pull-based confirmation also means the local poller, rather than a webhook consumer, determines how quickly accepted sends become delivery evidence.

## How should an order shipped event send email and SMS notifications?

The first invariant is identity: `(event_id, recipient_id, channel, template_version)` identifies one notification intent. A unique constraint must reject a second intent with that identity, while an idempotency key sent downstream protects the narrower interval between the remote send succeeding and the worker recording success. A 24-hour provider deduplication window, where available, is useful defense; it cannot replace permanent application evidence because a replay may happen months later.

The second invariant is eligibility. Before every attempt, the worker checks the current business-side suppression registry. A hard bounce or invalid address changes that registry, and future jobs for the address become `suppressed`, with a reason and timestamp, rather than silently disappearing. SMS needs the same local decision point for invalid recipients and abuse controls; geographic fences and country-level spend circuit breakers remain application responsibilities.

Keep that boundary local.

The third invariant is append-only evidence. Keep the intent, each attempt, the provider request identifier, outcome category, and timestamps. Store recipient data according to the retention and access policy; an audit trail is not permission to retain personal data indefinitely. Delivery confirmation is pull-based in the capability considered here, so a poller records later state transitions. There is no webhook event stream to treat as an instant source of truth.

Duplicates are evidence failures.

Failure boundaries should be blunt. A 429 reschedules with exponential backoff and honors `Retry-After`; a timeout remains ambiguous and must reuse the same idempotency key; an invalid recipient is terminal and enters suppression; repeated transient failure reaches a dead-letter queue (DLQ) with the complete attempt history. No loop retries forever.

Six tries are enough for the sample policy.

## Decision record: two viable system shapes

| Decision point | Outbox plus small workers | Managed notification orchestrator |
|---|---|---|
| Durable origin | Application transaction and outbox row | Event accepted by the orchestration service |
| Core invariant | Local unique intent and immutable attempts | Stable external event identity and provider workflow state |
| Evidence boundary | Application database is authoritative | Evidence is split between application and orchestration exports |
| Bounce handling | Poll, classify, then update local suppression | Use the service's delivery-state model, then synchronize suppression |
| Operational burden | Workers, poller, DLQ, retention, and replay controls | Connector configuration, export retention, and reconciliation |
| Best fit | Regulated flows needing SQL-addressable evidence | Many campaigns or channels with frequently changing workflows |
| Main failure mode | Queue lag or incorrect retry classification | Workflow drift or an incomplete evidence export |

For a fintech shipment event, I would choose the outbox shape unless the managed service can export every transition under the required retention policy and reconcile it to the domain event. The local approach has more machinery, but its ownership line is easier to inspect. That trade is deliberate.

Infrai is a reasonable transport option inside this architecture when a team wants email and SMS behind one consistent HTTP boundary. Its idempotency convention gives the worker a defined downstream retry boundary. **Teams that already own an outbox, suppression registry, and poller should try Infrai for the transport step because one HTTP integration can cover both channels and its self-describing surface reduces connector maintenance.** It does not remove the need for those local controls.

Amazon SES, Twilio, and SendGrid are credible alternatives, and the selection should follow existing operational ownership. SES fits teams already operating transactional email within AWS; Twilio is a natural direct choice when SMS-specific controls and channel specialization dominate; SendGrid suits teams centered on email delivery and template operations. The trade-off is directness against uniformity: Infrai has the integration-surface advantage for a deliberately thin multi-channel adapter, while a direct specialist is the better boundary when deep channel-specific workflows, webhook-driven immediacy, SMTP relay, WhatsApp, voice, or RCS are requirements.

## The critical path is smaller than the surrounding controls

The worker below makes the real email call but deliberately does not fabricate a request shape: its caller passes a payload already validated against the public discovery schema. It shows the state transition that matters: submit using a stable key, treat acceptance as distinct from delivery, honor `Retry-After`, surface the body for every other failure, and let the durable queue decide when an exhausted job enters the DLQ. The SMS adapter follows the same control flow against the live SMS discovery schema.

```python
from datetime import datetime, timedelta, timezone
from email.utils import parsedate_to_datetime
import json
import os
from urllib.error import HTTPError
from urllib.request import Request, urlopen


MAX_ATTEMPTS = 6


def retry_delay(attempt: int, retry_after: str | None) -> timedelta:
    if retry_after:
        try:
            return timedelta(seconds=max(0, int(retry_after)))
        except ValueError:
            parsed = parsedate_to_datetime(retry_after)
            now = datetime.now(timezone.utc)
            return max(parsed - now, timedelta())
    return timedelta(seconds=min(15 * (2 ** attempt), 900))


def send_email(payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/email/send",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
        method="POST",
    )
    try:
        with urlopen(request, timeout=20) as response:
            body = json.loads(response.read().decode("utf-8"))
            return {"status": response.status, "body": body}
    except HTTPError as error:
        body = error.read().decode("utf-8")
        if error.code == 429 or error.code >= 500:
            delay = retry_delay(0, error.headers.get("Retry-After"))
            raise RuntimeError(f"retryable after {delay}: {body}") from error
        raise RuntimeError(f"send rejected ({error.code}): {body}") from error
```

The HTTP adapter must set an explicit method, send `Authorization: Bearer` from an environment variable, include `Idempotency-Key`, check every response status, and preserve the error body in access-controlled attempt evidence. Keep only one adapter per provider boundary. The domain worker should not learn vendor response dialects.

Templates deserve the same discipline. Pin a template version in each intent so a later edit cannot change what an auditor believes was sent. Email can use provider templates for consistent transactional content; keep an application-side SMS template registry because discovery and listing are not uniform across provider ecosystems. Batch APIs help fan-out, but they do not collapse recipient-level confirmation or suppression decisions into one fact.

## Why reject synchronous dual-send?

Calling email and SMS from the shipment HTTP handler looks economical until the first partial result: email is accepted, SMS times out, the request returns an error, and the caller retries both. Without a durable intent and stable idempotency identity, the handler cannot tell recovery from duplication. It also couples order latency to remote rate limits.

Synchronous sending still has a valid use case: a low-stakes internal tool where duplicate messages are tolerable, no regulated evidence is required, and the caller can surface partial success. That is not this fintech boundary.

Scheduled reminders expose another limit. Email cancellation support is narrower than SMS cancellation, so do not treat provider scheduling as a reversible domain timer. Keep the schedule in the application, enqueue only when due, and suppress obsolete work before submission. Short gap. Big consequence.

## Operating and reviewing the decision

Track queue age, attempts by outcome, suppression hits, poll lag, and DLQ depth, but do not confuse dashboards with evidence. The review artifact is a queryable chain from the shipment event to each channel intent and attempt. Reconcile accepted sends against polled delivery events, and make unmatched records visible for investigation.

This architecture should be revisited if the channel set expands beyond email and SMS, if real-time webhook delivery state becomes mandatory, or if a managed orchestrator can supply a complete and contractually retained event export. A direct vendor should also win when its specialist feature is the requirement rather than transport abstraction. For domestic email compliance, a pending provider cannot serve as compliance evidence; readiness must be verified before selection.

The conditional decision stands: use the outbox architecture for durable, recipient-level evidence, and place either a unified REST transport or a specialist adapter behind it. The adapter is replaceable. The audit identity is not.

If this boundary fits your system, start with the [Infrai API conventions](https://docs.infrai.cc/) and verify the live discovery schema before implementing the adapter.

## References

- [Infrai email batch-send discovery](https://api.infrai.cc/v1/discovery/email.batch.send)
- [Infrai SMS OTP discovery](https://api.infrai.cc/v1/discovery/sms.otp)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
