# Phone Verification Evidence: Backend Retention for SMS OTP Login and Contact Routing

A phone verification flow's bill has several terms: attempted SMS deliveries, application and queue work, retained challenge records, and the engineering time spent reconciling ambiguous sends. For a B2B SaaS contact form that routes transaction-alert questions to the right support queue, begin by bounding delivery attempts and evidence retention together. Otherwise, a convenient resend button quietly becomes a second message producer with a longer-lived audit trail than the form itself.

Short answer: make the backend own the SMS OTP resend deadline, commit each authorized send with a durable outbox record, and retain a small routing receipt separately from the short-lived verification secret. A Next.js login or contact-form interface can display the countdown; the backend must decide whether another attempt is allowed. Keep transaction-alert delivery independent of this verification workflow.

Consider an explicitly hypothetical workload: 10,000 challenges in a planning interval, each with one initial SMS and two permitted resends. That bounds application-authorized send intents at 30,000 before considering transport retries. One accidental extra intent per challenge adds 10,000, a one-third increase over that ceiling. In this planning model, outbound attempts are the dominant variable because the model assumes their aggregate charge exceeds the storage and worker charges; confirm that assumption against actual invoices before using it for a budget. No unit price is needed to expose the amplification.

Count attempts first.

## How should backend phone verification retain SMS OTP evidence after login?

The useful entry point is evidence minimization: decide which facts a support worker needs after verification, then keep the transient authentication machinery out of that record. A form about a missing transaction alert might carry an authenticated tenant ID, an issue category such as `transaction_alert`, and a user-entered message. Its routing receipt can record the selected queue, the routing-rule version, and the verification outcome without becoming a second copy of the SMS message or its code.

Phone possession alone should not choose the tenant or grant access to transaction details. In this design, authenticated tenant membership determines the tenant boundary, while the phone challenge confirms possession for the particular form submission. A successful challenge does not authorize an arbitrary account lookup. Login has its own session-issuance policy; a transaction alert has its own delivery policy. Reusing a transport adapter is a reasonable integration shortcut, but merging those three authorization decisions creates an expensive support problem when somebody asks what a successful code actually proved.

Use four logical records, with retention policies that reflect their different purposes:

| Record | Proposed contents | Why keep it | Boundary or trade-off |
| --- | --- | --- | --- |
| Challenge | Opaque ID, purpose, bound tenant and destination, protected code verifier, expiry, attempt counters | Decide whether a submitted code or resend is valid | Delete secret material after expiry or consumption; delayed code inspection becomes unavailable |
| Send intent | Intent ID, challenge ID, destination reference, dispatch state | Recover authorized work across worker restarts | Retain while delivery is unresolved; avoid duplicating the form body here |
| Routing receipt | Submission ID, tenant ID, queue, rule version, verification outcome | Explain why support received the form | Keep for the declared support investigation window; omit the OTP |
| Aggregate counters | Attempts, suppressed resends, dispatch age, routing outcomes | Detect load and integration regressions | Aggregation sacrifices individual event reconstruction |

These are proposed schema boundaries, not claims that a particular database supplies the required guarantees. The challenge update and outbox insertion need a shared atomic commit. An object store can hold larger form attachments if the application needs them, but storing an attachment and writing its routing pointer introduces another consistency boundary: define when the submission becomes visible and how unreferenced objects are collected. Do not treat an uploaded object as proof that routing succeeded.

The receipt is small by design.

The free-text support message belongs under the support record's access and deletion policy, not in every delivery log.

## Bound retry amplification at the commit boundary

The change that moves the dominant term is to reject excess send intents before the transport adapter runs. Disabling a button helps the interface; it cannot serialize requests from two browser tabs, and a timeout can leave the caller unsure whether its first request committed. Require a caller-generated operation ID for each deliberate click, reuse it when retrying that operation, and return the committed result for a matching duplicate. Bind that ID to the challenge and operation payload so that it cannot silently identify different work.

Here is a deliberately narrow Python example of the resend admission decision. The values are illustrative application settings: a 300-second challenge lifetime, a 30-second cooldown, and three total send intents including the initial one. They are neither regulatory requirements nor measured optima. The caller supplies backend time, after loading the challenge under the application's transactional concurrency control.

```python
from dataclasses import dataclass, replace


@dataclass(frozen=True)
class Challenge:
    expires_at: int
    resend_at: int
    send_intents: int
    consumed: bool = False


@dataclass(frozen=True)
class Admission:
    state: Challenge
    enqueue: bool
    retry_after: int
    reason: str


def admit_resend(state: Challenge, now: int) -> Admission:
    # The initial challenge transaction already reserved one send intent.
    if state.consumed:
        return Admission(state, False, 0, "consumed")
    if now >= state.expires_at:
        return Admission(state, False, 0, "expired")
    if state.send_intents >= 3:
        return Admission(state, False, 0, "send_limit")
    if now < state.resend_at:
        return Admission(state, False, state.resend_at - now, "cooldown")

    updated = replace(
        state,
        resend_at=now + 30,
        send_intents=state.send_intents + 1,
    )
    return Admission(updated, True, 30, "accepted")
```

This function has no persistence and sends no SMS. Its integration contract is the consequential part: first validate the caller, purpose, and challenge binding; resolve an existing operation ID; lock or conditionally update the live challenge; apply the transition; then atomically persist the updated state, operation result, and outbox intent. Commit before returning acceptance. Without that contract, two requests can each observe two previous intents and both authorize a third.

The failure mode is **duplicate admission**. Test it with concurrent callers, not merely a sequential button test.

An outbox makes authorized work recoverable, but it does not establish exactly-once delivery across a remote sender. A worker can submit a message, lose the acknowledgment, and restart with an unresolved intent. Reuse a stable downstream idempotency key where the transport contract supports one, and document what that contract covers; otherwise classify the result as ambiguous and apply a bounded reconciliation policy. Blindly replaying the intent can turn the application ceiling into an inaccurate billing forecast.

Never equate “queued” with “received.”

For the countdown, return the backend's current time and next eligible resend time, and derive a remaining duration for display. On refresh or a new attempt, fetch authoritative state again. Avoid making a browser wall-clock timestamp the admission decision. Once the challenge expires, the response should communicate that a new challenge is required rather than display a countdown that implies the old one can still be used.

Keep code verification bounded separately from resend admission. Define expiry, single consumption, a verification-attempt limit, and a protected verifier instead of a plaintext-code log. Resending must not reset the guessing budget. The example intentionally does not implement code generation, verifier storage, session issuance, or destination-level abuse controls; those remain required integration work, and challenge-local limits alone do not bound an attacker creating fresh challenges.

## Integration effort includes proving the boundaries

The smallest credible integration has one challenge owner and one outbound adapter, with separate purposes for contact verification and transaction alerts. For US and EU deployments, explicitly record where the destination, challenge, outbox, and support receipt are processed. A region selector does not by itself establish a legal retention requirement. Choose and document the applicable policy with the people responsible for that deployment, then ensure deletion and access controls follow the records through workers and backups.

There is a concrete trade-off here. A single transactional store for challenge state and outbox reduces the number of commits the team must coordinate; splitting them across independently committed systems requires a recovery protocol and more failure testing. If the primary decision axis is integration effort, count that protocol, operational ownership, and migration work alongside the adapter code. An attractive SDK method is a poor measure of completed integration.

Test the edges that change either the bill or the queue: two simultaneous resends, a request retried after a lost response, expiry exactly at the admission boundary, a consumed challenge reused for a different submission, and an acknowledged send whose worker crashes before marking completion. Also test that an authenticated tenant cannot redirect a submission into another tenant's queue by changing the form payload. The acceptance criterion is the persisted decision and resulting work, not just a disabled button.

Deploy the routing-rule change separately from the transport change when possible. Record the rule version in the receipt so that a queue shift can be explained after deployment. A staged rollout can compare queue assignments and suppressed resend counts, while a rollback must continue honoring existing operation IDs and challenge deadlines. Resetting those records during rollback would change the safety properties of requests already accepted.

Measure intent count, dispatch attempts, unresolved-intent age, verification failures, and queue assignment failures. Keep phone numbers and codes out of metric labels. Track the difference between accepted intents and transport submissions because that gap distinguishes worker retry amplification from people deliberately pressing resend.

Email evidence needs its own interpretation. RFC 6376 specifies DKIM signing and verification; it does not make a signed message proof that a person read it. Apple's Mail Privacy Protection guide describes protection that prevents senders from seeing whether a message was opened. An email-open signal is an unsuitable substitute for an explicit verification outcome or a support-routing receipt. Delivery evidence and user action answer different questions.

## What disappears when the retention window closes?

Set retention from investigation needs, with shorter-lived secrets and longer-lived, minimized receipts. For illustration, a team could choose a seven-day routing-receipt window for this workflow; that is a planning choice requiring justification, not a US or EU standard. At a constant receipt arrival rate, reducing a 30-day window to seven days reduces the active receipt population to 7/30 of its former size, assuming deletion runs on schedule and excluding backups and unresolved work. It does not reduce the SMS attempts already authorized.

Deletion needs a completion definition. Expiring a challenge must prevent further authorization immediately, even if physical cleanup runs later. Removing the active row must not leave its code or full destination in worker payloads, diagnostic exports, or unrestricted backups. If an unresolved intent still needs a destination, give that exception an owner and a deadline instead of silently extending every challenge's lifetime.

Stop retaining plaintext OTPs, duplicate delivery bodies, and per-event detail once its declared investigation window closes. Preserve only the justified support receipt and aggregate operational evidence. The cost is specific: after detailed events expire, an old complaint may be explainable only as a recorded queue decision, not as a reconstruction of every dispatch attempt. That loss of forensic detail is acceptable only when the support policy says so. Budget the evidence you need before collecting evidence you cannot defend keeping.

## References

The protocol and privacy claims above are grounded in the following primary sources. The workload, thresholds, schema, and retention windows are explicitly illustrative design choices.

- RFC 6376, DomainKeys Identified Mail: signing and verification semantics for email.
- Apple, Mail Privacy Protection guide: limits on interpreting email-open observations.

## Further reading

- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
