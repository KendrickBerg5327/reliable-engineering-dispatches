# What I Require from Email Service: Password Reset and Welcome Deliverability

TL;DR: I would choose the HTTP email service that passes a production-like trial on the marketplace's own sending subdomain and exposes suppression and bounce events through an authenticated, replayable interface; I would not declare a universal winner from a feature matrix. For password resets, welcome messages, and new-order alerts, the decisive constraint is integration effort across the whole failure path, not the apparent ease of the first send call.

This architecture decision keeps the application API-first and excludes SMTP from the application boundary. The narrowest acceptable integration has one outbound adapter, one durable delivery ledger, and one inbound event consumer. It must let an operator answer a seller's plain question -- "Where is my order email?" -- without searching transient request logs or trusting a provider dashboard as the system of record.

## What should an email service prove for password reset and welcome deliverability?

A marketplace order is durable business state; an email is a fallible projection of that state. I therefore keep `order_id`, recipient, message purpose, template revision, provider message identifier, attempt state, and the last accepted event in a ledger controlled by the marketplace. The email provider may retain its own history, but that history is evidence rather than ownership. This distinction matters during migration and during the much more ordinary case where an API request times out after the remote side accepted it.

The first invariant is that creating order `ord_18472` can enqueue at most one logical `seller_new_order` notice. The second is that a retry may produce another transport attempt but cannot silently create another business notification. The third is that a known hard-bounced or complained-about address is suppressed before a new attempt. The fourth is that a reset message and a welcome message use separate streams or classifications from seller alerts, so investigation and policy changes do not collapse unrelated traffic into one bucket. These are application rules; buying an email API does not create them.

Failure boundaries deserve names. A client timeout is **ambiguous acceptance**. An HTTP success followed by no event is an **observation gap**. A duplicate callback is **at-least-once feedback**. A bounce arriving after a retry is event reordering. A valid order sent to a previously suppressed address is a policy breach, not a delivery error. Naming the modes makes the trial executable instead of subjective.

Timeouts are ambiguous.

Consider the concrete `ord_18472` path. The order transaction commits, an outbox worker reserves the logical notice, and the adapter submits it with that logical identifier as its idempotency key. If the connection closes before the response arrives, the only defensible ledger state is `ambiguous`: the remote service may have accepted the message, while the worker has no remote identifier to prove it. A second worker must see the reservation and stop. The reconciler then waits for correlated event evidence or uses a supported lookup before authorizing another transport attempt. If neither mechanism exists, the candidate has shifted duplicate risk into marketplace code, and that cost belongs in the selection record. The same sequence applies to a password reset, except that token lifetime makes delay operationally sharper; a welcome message is usually less urgent, but duplication is still visible to the recipient. One timeout therefore tests four boundaries at once: transaction-to-outbox durability, uniqueness in the ledger, remote idempotency behavior, and the completeness of feedback. A happy-path API demo tests none of them.

Keep that gap visible.

The sending domain also has to be under organizational control. Domain authentication is verified through DNS evidence, while DMARC gives domain owners a published policy and aggregate reporting mechanism; RFC 7489 is precise about that scope and does not promise inbox placement. A dedicated subdomain can separate transactional identity and operational changes from other mail, but isolation still does not excuse poor recipient data or weak suppression handling. No service can guarantee delivery to an inbox.

## Decision record: compare the work, not the brochure

I score a candidate only after wiring a thin adapter and event consumer in a disposable environment. The comparison is deliberately about boundaries that the team must own. A checkmark without a tested artifact counts as unknown.

| Decision surface | Acceptable evidence | Integration cost signal | Rejection condition |
|---|---|---|---|
| HTTP submission | Documented authenticated request plus a stable remote identifier | One adapter maps the internal envelope to the remote request | SMTP is required in the application path |
| Domain control | DNS records can authenticate a marketplace-controlled subdomain | DNS ownership and rotation have named owners | Shared identity is the only practical setup |
| Suppression | Query or event evidence distinguishes permanent failures and complaints | Local pre-send check and reconciliation job are feasible | Suppression state cannot be exported or observed |
| Bounce feedback | Authenticated events carry an identifier that maps to an attempt | One idempotent consumer handles duplicates and reordering | Events cannot be correlated to a submitted message |
| Operational history | Events can be retained in the marketplace ledger | Support can investigate without dashboard-only access | Required evidence expires before the audit window |
| Change isolation | Reset, welcome, and seller-order traffic can be classified separately | Policy and rollout changes have a contained blast radius | Every message shares one undifferentiated stream |

This table does not produce a durable global ranking. It produces a local answer: the best service is the candidate that clears every rejection condition with the least application-owned glue, then remains understandable under timeout, retry, suppression, and delayed-event tests. A candidate with more knobs can lose because each knob needs configuration ownership, deployment review, monitoring, and an exit plan.

Three limits keep the exercise honest. Run it with the actual domain-control process, not a personal test domain. Use controlled recipient addresses and documented test facilities rather than intentionally harming sender reputation. Retain no more recipient data in the evaluation ledger than the investigation requires.

## Critical path in Python

The adapter below is intentionally generic. It shows where the marketplace owns idempotency, suppression, correlation, and state transitions; `EmailApi` and `Ledger` are ports whose concrete implementations belong at the infrastructure boundary. The ledger must enforce uniqueness for `(order_id, purpose)`, because a read followed by a write is not enough under concurrent workers.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Envelope:
    logical_id: str
    recipient: str
    template: str
    variables: dict[str, str]


class Ledger(Protocol):
    def reserve(self, order_id: str, purpose: str) -> str | None: ...
    def is_suppressed(self, recipient: str) -> bool: ...
    def record_accepted(self, logical_id: str, remote_id: str) -> None: ...
    def record_ambiguous(self, logical_id: str) -> None: ...


class EmailApi(Protocol):
    def send(self, envelope: Envelope, idempotency_key: str) -> str: ...


def notify_seller(
    order_id: str, seller_email: str, api: EmailApi, ledger: Ledger
) -> str:
    logical_id = ledger.reserve(order_id, "seller_new_order")
    if logical_id is None:
        return "already_reserved"
    if ledger.is_suppressed(seller_email):
        return "suppressed"

    envelope = Envelope(
        logical_id=logical_id,
        recipient=seller_email,
        template="seller-new-order-v3",
        variables={"order_id": order_id},
    )
    try:
        remote_id = api.send(envelope, idempotency_key=logical_id)
    except TimeoutError:
        ledger.record_ambiguous(logical_id)
        return "reconcile_before_retry"

    ledger.record_accepted(logical_id, remote_id)
    return "accepted"
```

The short branch is the dangerous one. After a timeout, the worker does not assume failure and fire again; it marks an ambiguous attempt, then a reconciler uses the provider's supported lookup or subsequent event evidence before retrying. If the chosen API offers no useful idempotency behavior or lookup, the adapter must expose that limitation and the trial must measure the duplicate risk rather than hiding it.

Inbound events need equivalent discipline. Verify authenticity before parsing business fields, store the raw event under a retention policy, deduplicate on the event identifier when one is supplied, and make state transitions monotonic where possible. A delayed `delivered` event must not erase a later complaint merely because callbacks reached the consumer out of order. For webhook signatures, the exact verification algorithm and replay window are provider-specific and must come from that provider's current documentation; a generic sample would be false precision.

## Trial plan and operating evidence

A useful trial is small but adversarial. I use three message purposes -- password reset, welcome, and seller new order -- because their urgency, templates, and investigation paths differ, then exercise five conditions: normal acceptance, a client-side timeout, a duplicate event, an out-of-order event, and a recipient already present in suppression state. That is 15 cases before domain and deployment checks. The number is modest; the coverage is not.

For each case, capture the logical identifier, remote identifier when available, API outcome, event timestamps, final ledger state, and operator action. Avoid treating opens as proof of delivery or human attention. The hard evidence is the chain from a business event to API acceptance and then to delivery or failure feedback, with gaps left visible.

Deployment should start with one low-risk classification and explicit rollback criteria. Monitor acceptance latency, ambiguous attempts, hard bounces, complaints, suppression hits, event lag, unmatched callbacks, and duplicate callbacks as separate signals; a single "delivery rate" compresses distinct failure modes and slows diagnosis. Alerts should point to a runbook that explains reconciliation and suppression review without asking an operator to resend blindly.

Cost belongs in the record, but price per message is rarely the dominant term for this decision. Count DNS and security review, adapter code, event ingestion, data retention, on-call investigation, compliance review, and migration work. An apparently quick API that leaves event correlation or suppression export to bespoke code transfers effort into the least visible part of the lifecycle.

## Rejected option and the condition that changes the decision

I reject direct SMTP submission from each application service for this marketplace. It distributes credentials and retry behavior, encourages inconsistent headers and templates, and leaves every caller to solve correlation between an order and later delivery evidence. The application needs an HTTP boundary with one governed adapter, even if a downstream service uses SMTP internally.

SMTP remains valid when a mature internal mail relay already centralizes authentication, queueing, suppression policy, observability, and message identifiers, and application teams are required to use that relay. In that environment, replacing the relay can increase integration effort without improving the business boundary. The condition is concrete: the relay must expose the same durable evidence and policy controls demanded here.

**Decision:** choose no service until it passes the domain-controlled trial and every rejection condition in the table. Among those that pass, select the one requiring the least custom code to preserve the marketplace ledger, reconcile ambiguous sends, consume authenticated feedback, and export suppression state. Revisit the decision when traffic classifications, retention duties, domain ownership, or the internal relay boundary changes.

## References

- Amazon Simple Email Service Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://www.rfc-editor.org/rfc/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://www.rfc-editor.org/rfc/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://www.rfc-editor.org/rfc/rfc7489
- NIST Digital Identity Guidelines, out-of-band authenticators: https://pages.nist.gov/800-63-4/sp800-63b.html#out-of-band
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
