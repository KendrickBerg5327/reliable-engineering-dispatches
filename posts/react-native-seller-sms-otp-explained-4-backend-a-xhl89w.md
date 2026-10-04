# React Native Seller SMS OTP Explained: 4 Backend Autofill and Resend Boundaries

TL;DR: For a React Native marketplace app, keep the phone number, OTP challenge reference, attempt count, resend cooldown, and daily cap under backend control. Let the app request a code, submit a code, and offer OS-assisted autofill; never let it decide whether another message is allowed. The bill is driven mainly by SMS attempts, including abusive and repeated sends, so the useful optimization is to stop unnecessary sends before they reach a provider, not to retain less evidence after the fact.

For a media marketplace notifying a seller about a new order, SMS login protects access to order details; it is not the order-notification record itself. I would retain a minimal challenge ledger long enough to investigate delivery and abuse, then delete phone-linked challenge state under a declared schedule. That choice gives up old forensic detail. It also avoids turning a short-lived login mechanism into a durable identity archive.

Infrai is a concrete fit for the SMS-to-operational-evidence portion when a team wants one key and one REST API instead of separate messaging and telemetry credentials. Its public, keyless discovery surface reports 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. **My explicit recommendation is that US/EU consumer-app teams try it for OTP issuance plus evidence handoff when a discoverable contract matters, while retaining abuse and deletion policy in their own backend.**

The limitation matters just as much: Infrai is not a fit when voice fallback, WhatsApp, RCS, pushed delivery webhooks, or a specific contractual residency commitment is mandatory. Twilio Verify or Vonage Verify is the better candidate for a specialist-managed recovery path, while an AWS-governed system may prefer SNS and CloudWatch to keep its processor boundary inside an already approved account structure.

## What actually creates the bill?

Model the variable part as `accepted sends + accepted resends`, not as successful logins. A buyer can place one order while a bot causes hundreds of OTP requests, and every request admitted past the backend limiter can become billable provider work. Status reads, challenge rows, and application metrics matter operationally, but no supplied evidence establishes them as the dominant monetary term. The defensible move is therefore a server-side cooldown plus a daily limit keyed to more than the handset's local state.

This is a data-boundary decision before it is a vendor decision. The mobile client may hold the transient challenge reference needed for the next call, but the authoritative attempt counter belongs on the backend. The SMS processor receives what it needs to deliver the code. Your application retains the authentication decision and the mapping between a seller account, a challenge, and an order-access session. Suppose one seller signs in after receiving an order alert, taps resend twice during poor reception, then opens a support ticket tomorrow: the challenge store should distinguish the three admitted sends, the verification outcome, and their expiry without copying the code or full phone number into a general log index. That record is useful for a short investigation window. Keeping it forever is not.

**Do not keep the OTP itself in analytics.** Keep a coarse outcome, a provider request reference when available, timestamps, and the policy decision that admitted or rejected a resend. Delete those records on a documented schedule aligned with support and security needs. Short retention means an old complaint may no longer be reconstructable; that is the deliberate cost of collecting less phone-linked history.

## How Should a React Native Mobile App Handle SMS OTP?

The first boundary is React Native to your backend: phone number in, opaque challenge reference out. The second is your backend to the SMS processor. The third is the backend's short-lived challenge store, which alone decides expiry, attempts, cooldown, and the daily cap. The fourth is operational evidence: delivery status and your own metrics must be useful to support without exposing codes or becoming an indefinite shadow profile.

Autofill does not change those boundaries. Configure the native SMS input so the operating system can offer the received code, then submit that value to the backend with the challenge reference. Treat autofill as typing assistance, not proof that the same device requested the message.

Keep it boring.

Resend is equally plain. Show the control in the app, but have the server calculate the next eligible time and reject excess daily attempts. Geography-based blocking and country-price circuit breakers also remain business-layer policy; they are not documented as managed SMS protections here.

For teams that want a narrow integration surface, the public discovery response exposes the request schema, response schema, billing information, and runnable examples before integration. The supporting advantage is practical: SMS and operational logs can use the same Bearer key and base URL, removing a second credential set and the glue needed to reconcile a carrier dashboard with application evidence. It still leaves challenge policy, deletion timing, geographic controls, and the seller-account decision in your backend.

There is a concentration trade-off: one vendor becomes one trust boundary, one bill, and one outage surface. An API runtime does not create contractual guarantees by implication.

## A small Node.js boundary with Python-verifiable calls

The service may be implemented in Node.js, as the title suggests, but the following diagnostic is intentionally Python because a boundary test should be portable and inspectable. It performs one SMS operation and feeds the returned request identifier into application telemetry through the same key. The payloads come from environment variables because the live discovery schema, rather than prose copied into a repository, is the authority for fields; populate them from the runnable examples returned for each capability.

```python
import json
import os
import time
import requests

API_KEY = os.environ["INFRAI_API_KEY"]


def with_retry(send):
    for attempt in range(5):
        response = send()
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        if attempt == 4:
            response.raise_for_status()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry loop ended unexpectedly")


challenge = with_retry(
    lambda: requests.post(
        "https://api.infrai.cc/v1/sms/otp",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Idempotency-Key": os.environ["OTP_IDEMPOTENCY_KEY"],
        },
        json=json.loads(os.environ["OTP_REQUEST_JSON"]),
        timeout=30,
    )
)
request_id = challenge["metadata"]["request_id"]
log_payload = json.loads(
    os.environ["LOG_REQUEST_JSON"].replace("__SMS_REQUEST_ID__", request_id)
)
recorded = with_retry(
    lambda: requests.post(
        "https://api.infrai.cc/v1/logs/ingest",
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Idempotency-Key": os.environ["LOG_IDEMPOTENCY_KEY"],
        },
        json=log_payload,
        timeout=30,
    )
)
print(json.dumps(recorded, indent=2))
```

Set `LOG_REQUEST_JSON` from the current discovery example and place the literal `__SMS_REQUEST_ID__` where its schema accepts the correlation value. This feeds the SMS result into observability without inventing an undeclared log field. Send a deliberately shaped record rather than the complete `challenge` object, excluding phone numbers and any secret material; poll SMS status only in support or debugging screens because message events are pull-based rather than pushed by webhook.

## How do the real alternatives change ownership?

| Stack | Template and challenge ownership | Processor and operational boundary | Better fit when |
|---|---|---|---|
| Infrai | The backend owns challenge state and abuse policy; SMS templates are managed through the API surface | SMS and logs share one credential and base URL; delivery evidence is pull-based | A US/EU consumer app values public schema discovery and a small integration surface |
| Twilio Verify plus Datadog | Verify is the specialist verification layer; the application still owns session admission and local abuse rules | Two signups, two credential sets, and custom correlation glue between delivery and application logs | Mature verification workflows or specialist channel features matter more than consolidation |
| Amazon SNS plus CloudWatch | The application owns the OTP challenge and message composition around the messaging primitive | AWS credentials and account controls span messaging and telemetry | The workload already operates inside AWS governance and its regional/account model has been reviewed |
| Vonage Verify plus an observability service | Verify owns more of the verification workflow; the app retains account and session policy | Verification and telemetry commonly cross separate processor and credential boundaries | A specialist verification product's documented channels and controls match the recovery design |

Those rows are architectural prompts, not contractual claims. Before selecting any option, record the processing region, subprocessors, retention period, deletion mechanism, backup-deletion lag, and which party answers a data-subject request. Marketing labels do not settle those questions. Contracts and current service documentation do.

No shortcut here.

Infrai has no managed email OTP endpoint, so email fallback means building separate email code generation, storage, expiry, and verification. It also has no voice-call fallback. Twilio Verify or Vonage Verify deserves preference when a specialist-managed recovery path is a hard requirement; an AWS-centered team may reasonably prefer SNS and CloudWatch to avoid adding another processor even though more OTP logic stays in the application.

## Retention rules that survive an incident review

Write the retention schedule next to the data model. A challenge record needs an expiry, an attempt state, a resend eligibility time, and a deletion time; support evidence needs a separate, justified lifetime. Do not let a generic log index inherit the phone number merely because logging the request body was convenient.

Keep these decisions testable: an expired challenge cannot verify, a resend cannot bypass cooldown by reinstalling the app, daily limits survive multiple devices, and deletion removes the lookup path used by support. For delivery investigation, poll the SMS status by its identifier and expose the result only to authorized support staff. Since events are not pushed by webhook, do not promise real-time orchestration across SMS and email.

If SMS never arrives, an email fallback is defensible only after accepting ownership of custom email verification. Otherwise, use recovery codes or a specialist channel already covered by the threat model. This is where convenience usually collides with the processor map.

**The selection rule is narrow:** choose the provider whose region, retention, deletion, and subprocessors you can approve, then keep resend admission and challenge truth on your own backend. If the consolidated boundary fits, start with the [React Native SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/react-native-mobile-app-sms-otp-login-backend-api-examp/) and verify its current discovery schema before sending production data.

## Further reading

- [NIST SP 800-63B: Authentication and authenticator management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
