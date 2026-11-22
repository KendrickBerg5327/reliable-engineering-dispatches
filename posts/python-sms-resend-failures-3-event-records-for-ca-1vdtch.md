# Python SMS Resend Failures: 3 Event Records for Carrier Filtering

For fintech signup verification links, SMS event notifications create an operating bill that includes message attempts, delivery-observation work, retained evidence, and engineering time spent explaining resend failures or carrier filtering. Consider a hypothetical budget of 1,000 signups, each with one initial send and permission for one resend: initial delivery accounts for 1,000 attempts, and unconditional resending adds another 1,000. Message attempts are the dominant count in that simplified delivery budget; their monetary share still depends on the actual contract and the other operating costs.

Short answer: troubleshoot SMS event notification failures by verifying sender registration and signatures for the US/EU destination before resending, then use status and event polling to distinguish queued, delivered, failed, and carrier-rejected outcomes. Keep 3 application records: the send intent, the delivery observation, and the decision authorizing another attempt. This makes integration effort visible where it otherwise hides: in reconstructing a decision after the link has expired.

Change the authorization rule before trimming the archive. Requiring an explicit decision for each resend removes unconditional repeat attempts from the design, although it does not establish a measured reduction in failed deliveries or charges. The same distinction matters for telehealth login verification: a delivery outcome explains transport, while the application decides whether authentication remains valid.

Count decisions first.

Integration also includes credentials and accounting. A unified backend API with a single key and one bill can consolidate SMS, email fallback, and supporting storage under one credential and one billing relationship; a self-describing interface addresses a separate task, learning each operation's contract. Neither advantage replaces sender registration.

## How should SMS event notifications justify a resend after carrier failures?

A carrier rejection should send the investigation toward destination-specific sender registration, signature configuration, and the available rejection evidence. A queued observation establishes something different. Neither a delayed observation nor an HTTP timeout establishes that nothing was sent, so a worker that interprets either as permission for a fresh send can create a duplicate even while its local retry loop appears reasonable.

The accounting unit should therefore be an authorized operation. A transport retry reuses that operation's identity; an intentional resend needs a new authorization tied to the earlier attempt. SMS resend and cancel operations can support that workflow, but a cancellation request should be reconciled with subsequent status evidence, and revoking a verification link remains a separate application action. Delivery is not redemption.

| Application record | Minimum useful evidence | What it cannot prove |
| --- | --- | --- |
| Send intent | Signup reference, operation identity, destination market, sender configuration reference | That the provider accepted or delivered the message |
| Delivery observation | Provider message reference, observed outcome, observation time, retained rejection explanation where available | That the recipient used a valid link |
| Resend decision | Earlier attempt reference, authorization reason, decision identity, disposition | That a later network operation completed |

These are proposed application records, not a provider response schema. In particular, the sender reference should identify the configuration actually used, rather than point to a mutable configuration whose current contents silently replace yesterday's evidence. A durable object containing a log line preserves bytes; it does not settle which concurrent worker owned a resend decision. Put that authority in transactional application state and treat any private archive as supporting evidence.

Don't infer approval.

For US/EU routing, implement application-owned destination allowlists and country spend cutoffs alongside the resend policy. Registration is not a fraud budget. A valid sender configuration cannot decide whether a signup is allowed to consume another attempt.

## Read the delivery evidence before committing another decision

Start with a read rather than another send. This Python 3 example fetches status for an existing message, preserves the response without guessing its fields, and retries HTTP 429 with exponential backoff while honoring Retry-After. Set INFRAI_BASE_URL to the documented HTTPS API origin without a path, INFRAI_API_KEY to the API credential, and SMS_MESSAGE_ID to the existing message identifier. Configuration keeps the deployment address and credential out of a reusable investigation note.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlsplit
from urllib.request import HTTPRedirectHandler, Request, build_opener


class NoRedirect(HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        return None


origin = os.environ["INFRAI_BASE_URL"].rstrip("/")
parts = urlsplit(origin)
if (
    parts.scheme != "https"
    or not parts.netloc
    or parts.username is not None
    or parts.password is not None
    or parts.path
    or parts.query
    or parts.fragment
):
    raise ValueError("Configure an HTTPS API origin without a path")

message_id = quote(os.environ["SMS_MESSAGE_ID"], safe="")
request = Request(
    origin + f"/v1/sms/status/{message_id}",
    headers={
        "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
        "Accept": "application/json",
    },
    method="GET",
)
opener = build_opener(NoRedirect())

for attempt in range(5):
    try:
        with opener.open(request, timeout=20) as response:
            print(json.dumps(json.load(response), indent=2))
        break
    except HTTPError as error:
        reason = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(
                f"Status query failed: HTTP {error.code}: {reason}"
            ) from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                server_delay = float(retry_after)
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                if retry_at.tzinfo is None:
                    retry_at = retry_at.replace(tzinfo=timezone.utc)
                server_delay = (
                    retry_at - datetime.now(timezone.utc)
                ).total_seconds()
            delay = max(delay, server_delay)
        time.sleep(max(0.0, delay))
```

Only the configured API origin receives the Authorization header, and redirects are refused. The example is read-only, so it needs no write-idempotency key. Persist the interpreted observation against the send intent using the capability's response schema; successful retrieval is not an authorization to resend.

The difficult boundary comes when a decision reserves another attempt. If a worker stops before sending, the reservation remains unresolved; if it stops after acceptance but before recording the response, the application cannot infer failure from its missing receipt. A production dispatcher must recover reserved work, preserve the operation identity across transport retries, use the selected delivery contract's idempotency mechanism where supported, and reconcile provider evidence before authorizing anything new. An outbox can durably record work, but it cannot make the database commit and an external carrier delivery one atomic transaction. That is the integration work to estimate.

One possible application policy is to allow only one resend authorization for a given signup generation, enforced by a unique database key. That is a proposed limit, not a carrier rule. Link expiration and single-use redemption also belong to the application security design; OWASP's token guidance is relevant to that boundary, not evidence that a carrier delivered a particular message.

## Choose who owns the unresolved work

The provider comparison becomes more useful after defining the records, because the question is now concrete: which contract leaves the least additional work between an authorized signup link and an explainable delivery outcome? Evaluate a general SMS transport separately from a managed verification workflow. Replacing the latter may change the user interaction, which matters if the product requirement is specifically a link rather than a code.

| Candidate | Integration starting point | Boundary to resolve before choosing |
| --- | --- | --- |
| Amazon SNS | Evaluate message publishing for an application-owned verification flow, particularly where AWS integration already exists. | Include sender configuration, correlation with signup state, and delivery-evidence handling in the work estimate. |
| Twilio Verify | Evaluate its verification workflow if delegating verification orchestration is an actual requirement. | Confirm the required link interaction and destination requirements; a verification product is not automatically interchangeable with an arbitrary SMS sender. |
| Vonage Verify | Compare its verification workflow against the same application-owned records and signup interaction. | Validate the required channel and link behavior before assigning it responsibility for the existing flow. |
| Shared REST platform | Public, self-describing discovery supplies request and response schemas and runnable examples, so integrating a capability starts with reading its contract rather than learning a new SDK. One API key covers 295 routes across 20 modules. | Budget for a polling-based reconciliation worker and application-owned destination and spend controls. Use status and event polling for reconciliation, and SMS resend and cancel for authorized follow-up. A design requiring pushed delivery events needs a different integration contract. |

Infrai offers that self-describing REST interface alongside one API key and one bill across backend services. The discovery advantage concerns learning the contract. The second advantage is unified billing and single-key access: one credential can serve the signup dispatcher, email fallback, and supporting storage, while consolidated billing avoids reconciling separate service invoices. That multi-service breadth reduces the credential and accounting integrations the team has to maintain. It doesn't establish superior carrier delivery or a measured integration-time advantage.

Another contract detail worth inspecting is idempotency: the shared platform specifies the Idempotency-Key header and a default 24h deduplication window for applicable operations. Check the chosen capability's declaration before relying on it. That bounded window is a limit, not permanent duplicate suppression, so the application's operation record must outlive it when recovery policy requires longer evidence retention.

For the shared REST path, plan around a pull-based email/SMS workflow and application-owned email fallback verification. Keep scheduled fallback messages subordinate to the application's link-validity decision, since SMS cancellation and token revocation have different meanings. Maintain the template references and business-purpose accounting in the application itself. Scope this design to email and SMS; an SMTP relay or voice, WhatsApp, or RCS requirement calls for a different channel evaluation. US/EU sender checks also do not establish another jurisdiction's compliance.

This is a fit decision. If a team already owns link verification and AWS operations, start by evaluating the publishing path; if it wants verification orchestration, compare the two verification products against the exact interaction first. If inspecting schemas and consolidating backend credentials removes more work, the shared REST approach deserves a place on the shortlist. Require the same evidence from each candidate: the intended signup interaction, destination-specific sender configuration, and a recoverable record of every authorized resend.

## Retain decisions longer than repetitive observations

The retention trade-off is between reconstructing authority and preserving every transport detail. A proposed policy is to keep the three decision records through the application's investigation window, retain sender and template configuration versions with them, and compact repeated identical polling observations into a first-seen time, a last-seen time, and a count. Those are application fields, not promised response fields. Keep state transitions and the rejection explanation needed for troubleshooting; don't retain a redeemable verification link merely because the easiest logging call serializes the whole request.

Compaction loses detail.

After it, an investigator can establish which outcome was observed and how often it recurred, but cannot reconstruct every intermediate response or the exact interval between identical observations. Deleting raw responses also limits later reinterpretation if the application's parser was wrong. Choose the investigation window with the signup security and retention requirements in view, keep access to remaining evidence private, and make deletion part of the design rather than an emergency cleanup job.

The deliberate stopping point is repeated payloads and live verification secrets, while retaining enough authority and outcome evidence to explain why the application sent another message. When something goes wrong after that evidence expires, some carrier disputes and parser investigations will remain unresolved. That is the cost of the retention boundary; another resend cannot recover deleted history.

## Further reading

References for evaluating delivery contracts and application-owned verification:

- Amazon SNS SMS messaging: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify documentation: https://developer.vonage.com/en/verify/overview
- OWASP Forgot Password Cheat Sheet, including token handling: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo sender best practices, relevant to email fallback delivery: https://senders.yahooinc.com/best-practices/
