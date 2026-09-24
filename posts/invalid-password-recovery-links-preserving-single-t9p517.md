# Invalid Password Recovery Links — Preserving Single Use Under Support Pressure

A password recovery link that reports an invalid token should be treated as consumed, expired, superseded, malformed, or unverifiable, never as a reason to relax validation. In a customer-support system, store only a digest, consume the token atomically, use a deliberately limited expiry window, and issue a fresh link through the same abuse-controlled path when recovery fails.

**TL;DR:** Separate token presentation from token consumption, and separate both from account deletion. A successful password reset should invalidate existing sessions; an authorized GDPR deletion request should enter its own workflow and revoke every session before destructive work begins. The deciding constraint is bot and abuse resistance: an operator shortcut that resurrects a token, extends its deadline, or bypasses mailbox control creates an account-takeover path.

## Why does my password reset link say the token is invalid?

The browser only knows that a string arrived in a URL. The server must establish more: the string has the expected encoding, its digest matches one live record, its deadline has not passed, it has the intended purpose, and no competing request has consumed or superseded it. “Invalid token” is therefore a deliberately broad public response. OWASP recommends consistent messages and timing in recovery flows so attackers cannot use differences to enumerate accounts.

Several failures collapse into that response. A mail security scanner may follow a link before the person does if the application consumes on GET. A user may request two emails and open the older one. Two tabs can race. A replica can lag behind the primary that recorded consumption. URL rewriting can damage encoding, while logs or analytics can leak a bearer token.

Start with the race.

Consider the concrete sequence rather than the final error page: the customer requests recovery at 09:00, requests it again after assuming the first email was lost, and then opens the delayed first message. Under a newest-link-only policy, the first token is correctly superseded even if its printed expiry time has not arrived. In another case, a mail gateway performs a GET seconds after delivery; if GET consumes, the human sees an invalid token on the first deliberate click. A third case opens the same form in two tabs and submits both. Only the transaction outcome distinguishes these paths reliably, which is why a generic browser message and a precise internal state can coexist without contradiction.

Start debugging with correlation, not token inspection. Give each issuance a random internal record identifier; record `issued_at`, `expires_at`, `consumed_at`, and `superseded_at`; attach a correlation identifier to presentation and consumption events. Never log the raw link, query string, reset token, password, or session credential.

One boundary matters: rendering a form is not consumption. A GET may validate enough to show the form, but the atomic transition belongs on the password-changing POST. Otherwise previewers, accessibility tools, mail gateways, and double-clicks can burn the credential without changing a password.

Do not consume on GET.

## Decision record: invariants and failure boundaries

The recovery record is a short-lived capability with a narrow purpose. Exactly one password change may commit; expiry is checked by server time in the same authoritative transaction; issuing a replacement supersedes prior live records under a documented policy; successful consumption revokes all account sessions. Public callers receive one uniform failure, while internal telemetry records a bounded reason code.

Account deletion is a separate state machine. A reset can restore access so the holder can request erasure, but it must not delete data by side effect. Once deletion is authorized, revoke sessions and block new session issuance, then enqueue idempotent deletion work across data stores. That closes the gap in which an old session could act while deletion fans out.

| Option | Single-use boundary | Main failure mode | Valid use |
|---|---|---|---|
| Transactional row with conditional update | One authoritative commit | Replica reads can make prechecks misleading | Recovery records beside a transactional identity store |
| Compare-and-delete in a linearizable key-value store | Atomic key mutation | Retry and expiry semantics require care | High-volume ephemeral capabilities with a proven consistency contract |
| Signed self-contained token alone | Signature and deadline only | Replay remains possible until expiry | Repeatable claims where single use is unnecessary |

The third option is rejected here. Cryptographic validity does not record that a reset succeeded, so a self-contained token still needs server-side state when the requirement is one successful use. It is reasonable for safely repeatable assertions. A password change is not one.

This choice has limits. A transactional row is unsuitable when the identity store cannot provide an authoritative conditional write, and a linearizable key-value design carries an operational trade-off: its separate data lifecycle and retry semantics must be reconciled with the account database. The signed-token option reduces lookup dependence but gives up intrinsic single-use enforcement. None is universally better; the invariant selects the mechanism.

## The critical path belongs in one transaction

The generic example keeps the decisive validation and mutation together. A precheck may improve presentation, but it cannot establish single use.

```python
from datetime import datetime, timezone
import hashlib

def consume_reset(db, raw_token: str, new_password_hash: bytes) -> bool:
    now = datetime.now(timezone.utc)
    digest = hashlib.sha256(raw_token.encode("utf-8")).digest()

    with db.transaction() as tx:
        record = tx.select_reset_for_update(digest=digest)
        if record is None:
            return False

        usable = (
            record.consumed_at is None
            and record.superseded_at is None
            and now < record.expires_at
            and record.purpose == "password_reset"
        )
        if not usable:
            tx.record_safe_failure(record.id, occurred_at=now)
            return False

        if tx.consume_if_live(record.id, consumed_at=now) != 1:
            return False

        tx.update_password(record.account_id, new_password_hash)
        tx.revoke_all_sessions(record.account_id, revoked_at=now)
        tx.append_security_event(
            account_id=record.account_id,
            event_type="password_reset_completed",
            occurred_at=now,
        )
    return True
```

Compute the intentionally expensive password hash before entering this short transaction, then pass in the result. The commit must still bind token consumption, password replacement, session revocation, and the security event. If those writes span systems, use a durable handoff and deny refresh as soon as the account security version changes; do not promise immediate global revocation unless the session architecture enforces it.

The expiry rule above is `now < expires_at`. At equality, the token is expired. Test that exact boundary and avoid rounding timestamps differently between issuance and consumption. There is no universal duration: the window trades delivery latency against exposure after mailbox or link compromise. Measure mail arrival and completion time, document the chosen limit, and review failures without exposing account-specific timing.

The expiry window is a trade-off, not a badge of security.

## Debugging without creating an oracle

Take one failed attempt's correlation identifier. Check whether issuance completed, another issuance superseded it, consumption committed, or server time crossed the deadline. Then inspect concurrent requests and mail-preview traffic. Support needs bounded statuses such as `expired`, `already_consumed`, and `superseded`; the unauthenticated page does not.

Apply limits on both account and network dimensions, with allowance for shared networks. Reset-request responses should remain consistent whether an account exists, and delivery should be asynchronous so mail latency does not become an enumeration signal. Risk challenges can raise automation cost, but they do not replace throttling, token entropy, uniform responses, or monitoring.

Test the ugly paths. Race two consumption requests and assert exactly one password update. Present a token immediately before, exactly at, and immediately after expiration. Request a replacement, then try the first link. Replay after success. Verify every preexisting session can no longer authenticate, including refresh paths and long-lived support-console sessions. Confirm logs, traces, analytics, referrer headers, and error reports contain no raw token.

Count issuance attempts, delivery attempts, presentation failures by safe reason, successful consumption, and session revocation. Alert on ratio changes rather than treating every invalid link as an attack. A scanner burning tokens produces a different sequence from credential stuffing: presentation follows delivery, no human interaction appears, and no password-change commit occurs.

Evidence beats guesswork.

## Operational rule for support and deletion

Support may resend through the normal workflow, explain that only the newest link is usable, and identify whether a deletion case is blocked on restored access. Staff must not reveal whether an email has an account, mark an expired token live, paste a link into a ticket, set a temporary password, or exempt an account from controls without a separately audited process.

For GDPR erasure, recovery is only one possible authentication step. After authorization, move the account to a deletion-pending state that rejects new logins, revoke every session, and execute deletion as idempotent work with an auditable completion state. Retention duties and backup handling need their own policy; a reset-token table cannot answer them.

The decision is straightforward: retain state when single use is required, make consumption atomic, disclose less to unauthenticated callers than to operators, and give support a resend path instead of an override. An invalid link is an expected terminal state. The dangerous bug is allowing operational convenience to make it valid again.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
