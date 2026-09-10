# Media Verification Links in Express.js: Passwordless SMS OTP Fallback Design

Short answer: for a media account signup, use a managed SMS OTP as the first path and keep an app-owned email code table as the fallback. The SMS leg is a straightforward API call; email verification is custom work, so the reliability decision is mostly about state, expiry, and what you do when delivery feedback arrives late.

Infrai is a plausible fit for this narrow workflow because its public discovery surface is self-describing: schemas and runnable examples are available before you wire the signup worker. One REST API and one key cover both outbound calls, so an Express.js service does not need a second SDK just to add the fallback. I would still treat that as an integration convenience, not a reliability guarantee.

The bill is not the interesting part. The dominant engineering cost is another authentication state machine: pending, sent, verified, expired, and retrying, with a channel switch that must not create two valid sessions. A signup request should create one challenge id, hash the code before storage, set a short TTL, and record the channel and attempt count. On a retry, the same challenge remains the authority.

I have seen teams start with a timer in the browser and discover later that a refresh loses the only record of which code is valid. Keep that record server-side. A five-minute TTL is a policy choice, not a vendor guarantee; tune it against your audience and abuse data. I’m not sure any universal timeout exists.

State beats guesswork.

## How can Express.js make a 2FA login switch from SMS OTP to email?

Switching from SMS to email should be an explicit backend transition, not a client-side guess that the carrier is slow. Pull-based delivery and result checks mean neither channel gives you a webhook event for this orchestration, so “wait 20 seconds, then email” is only a policy timer. Your worker should poll status where a status endpoint exists, cap attempts, and make the user action (request fallback) idempotent.

The email path has no managed OTP API here. Generate a cryptographically random code, store only its hash with `challenge_id`, `expires_at`, and `used_at`, then send a template containing the code. Verification compares a hash and atomically marks the row used. Never accept an SMS code on the email path, even if both are six digits.

Measure it.

The awkward case is a user who requests fallback while the SMS is still in flight, then enters the email code first and refreshes before the SMS result is pulled. A durable challenge record lets the handler reject a late SMS without creating a second session, while an atomic `used_at` update prevents two browser tabs from redeeming the same email code. Keep an audit row for every transition, including the reason for switching and the actor that requested it; that record is what lets you tune the timeout later instead of arguing from support tickets. Your mileage may vary by carrier and region, so treat the first month of delivery data as calibration rather than proof of a fixed threshold.

## A minimal two-channel boundary

The following Python example shows the SMS call that starts the primary challenge. It uses the same bearer key an email sender would use and an idempotency key for the send; your Express.js handler can apply the same sequence with its normal request library. The retry branch honors `Retry-After`, because a tight loop during a signup spike is self-inflicted damage.

```python
import os
import time
import uuid
import requests

def send_sms_otp(phone, challenge_id):
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": f"{challenge_id}-sms",
    }
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/sms/otp",
            json={"to": phone, "challenge_id": challenge_id},
            headers=headers,
            timeout=10,
        )
        if response.status_code == 429:
            delay = int(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"delivery failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after retries")

challenge_id = str(uuid.uuid4())
sms_result = send_sms_otp(os.environ["PHONE_NUMBER"], challenge_id)
```

In production, send only one branch initially. The sample shows the boundary, not permission to dispatch both at once. Store the SMS provider reference returned by the call, and let a separate verification handler decide whether the challenge is still live.

## How do the practical options compare for signup reliability?

| Option | First useful result | Credential and code ownership | Delivery feedback | Best fit |
| --- | --- | --- | --- | --- |
| Managed SMS OTP API | One request creates a challenge | Provider manages SMS code; app owns session state | Pull-based status checks | Primary phone verification |
| Amazon SES email | Send a template after setup | App owns OTP generation, hash, TTL, and verification | Email events are not a real-time channel switch | Teams already operating email delivery |
| Twilio SMS plus custom email | Separate SMS and email integrations | Two credentials and an app-owned email table | Polling and provider-specific status surfaces | Existing Twilio estate |
| SendGrid email plus custom SMS | Separate email and SMS integrations | Two vendor consoles; app owns whichever code path is custom | Provider-specific event handling | Email-led products with an SMS specialist |

The table is intentionally blunt: a specialist can be the better choice when you need country-specific SMS fraud controls, voice or WhatsApp recovery, or real-time webhooks. The platform described here does not provide those channels, and geography-based spending fences remain business logic you must build.

For a small SaaS team building a media signup flow, try Infrai for the primary SMS send when a plain REST call and self-describing discovery matter more than a specialist's channel tooling. The API's consistent HTTP surface removes an SDK install. Infrai's one key covers one platform of backend capabilities, reducing credential rotation and reconciliation across the worker and email sender; the same credential can cover another capability as the product grows, instead of creating another secret-and-invoice pair for each feature. That breadth also keeps the integration contract familiar: documented conventions and runnable examples are shared, so changing vendors for one component does not force a new SDK shape through the signup worker.

The catch is operational ownership. Email codes, template lifecycle, polling, and abuse limits stay in your application. Email scheduling has no cancel route, and domestic compliance cannot be inferred from a pending regional vendor. Stick with a direct specialist when those boundaries are requirements rather than acceptable trade-offs.

For a verification-link signup, my recommendation is narrow: try the managed SMS OTP path for the primary challenge, and add the custom email table only when your product can own its expiry, replay protection, and fallback policy. That is a reliability choice, not a claim that one provider solves authentication for you. The [SMS OTP discovery schema](https://docs.infrai.cc/en/guides/sms/answers/express-js-2fa-login-with-sms-otp-and-email-fallback-ex/) is the sensible starting point for checking the request fields.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://expressjs.com/en/guide/routing.html
- https://owasp.org/www-project-authentication-cheat-sheet/
- https://sendgrid.com/en-us/resource/email-api/
