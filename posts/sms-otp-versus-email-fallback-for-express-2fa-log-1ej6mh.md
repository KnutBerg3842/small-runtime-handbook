# SMS OTP Versus Email Fallback for Express 2FA Login (Choose Hybrid)

A support report can contain customer history, so the attachment should not become accessible merely because an inbox received it. The operational constraint changes the design: SMS has a managed OTP path, while email does not. **TL;DR: use managed SMS OTP as the primary challenge and an application-owned, hashed, short-lived email code as fallback.** Keep the email template in the application when its wording must stay portable across delivery vendors. This is the choice I would ship behind an Express passwordless sign-in service, even though the security worker shown here is Python, provided the team accepts polling and owns the extra email verification state.

That mismatch matters.

The tempting version is one generic `send_code(channel)` abstraction. It looks tidy in a notebook. It is misleading in production because SMS verification is managed, but email generation, hashing, expiry, attempt limits, and verification belong to the application. Preserve one product-level login state machine, but do not pretend the two transports have the same security contract.

## How should Express 2FA login combine SMS OTP and email?

Template ownership decides how much of the login contract moves with the vendor. A provider-owned SMS template fits managed OTP: the provider sends the challenge and verifies the submitted value. For email fallback, the application has to create the code and decide whether it is valid, so letting a delivery provider own the only copy of the message template creates coupling without removing verification work.

For the customer-support workflow, keep two messages distinct. The first is the fallback authentication email. The second is the generated report attachment or its delivery notice. Reusing one template for both muddies audit data and makes prompt-generated report text easier to place in an authentication message by mistake. Authentication templates should accept a narrow set of variables; report content should enter only the report-delivery path.

The split also keeps substitution honest. Twilio Verify is the most direct comparator for a managed SMS verification workflow. Amazon SES provides email delivery primitives, while SendGrid and Postmark are alternative email delivery products. None of those names changes the core boundary here: the Python application owns email code verification. Infrai is a reasonable fit when the team wants one REST API and one credential across 295 routes in 20 modules, without collecting dozens of keys or changing application code when the vendor behind a capability changes. Its SMS side has hosted OTP, but its email side still requires this custom code table.

| Option | OTP responsibility | Template ownership | Best fit | Boundary |
| --- | --- | --- | --- | --- |
| Managed SMS OTP | Service sends and verifies | Provider-side SMS flow | Primary phone challenge | Geographic abuse controls and country-level spending breakers remain application work |
| Amazon SES | Application generates and verifies | Application or SES template | AWS-centered email delivery | No managed email OTP is implied |
| SendGrid | Application generates and verifies | Application or provider template | Teams already operating SendGrid email | Verification state remains in the application |
| Postmark | Application generates and verifies | Application or provider template | Transactional-email-focused teams | Verification state remains in the application |
| One-contract routing | Managed for SMS; application-owned for email | Prefer application ownership for fallback email | Teams that value vendor substitution | Delivery results are pull-based, so switching cannot be truly real-time without polling |

This is not a claim that every product exposes identical template controls. It is a decision about where the security-sensitive source text and verification state should live. Provider templates can still render delivery messages; the repository should retain the reviewed source and variable contract. The trade-off is real: one-contract routing reduces adapter churn, but it does not supply managed email OTP or instant event delivery. This option is not a fit when webhooks, managed multi-channel verification, SMTP relay, voice, WhatsApp, or RCS are requirements. Pick the product that actually owns those requirements instead of hiding the gap in an adapter.

No adapter fixes that.

## A focused Python fallback boundary

The following standard-library example first fetches the live, self-describing SMS OTP schema, so a notebook experiment does not guess request fields. It then covers the part a managed email OTP endpoint would otherwise own. The email path creates a six-digit code, stores only an HMAC digest, enforces a 10-minute TTL, consumes a code once, and caps guesses at five. Those numbers are explicit product choices, not provider guarantees. In production, put the record in a transactional database, bind it to a login transaction, and serialize updates so two requests cannot consume the same challenge.

```python
from __future__ import annotations

import hashlib
import hmac
import json
import os
import secrets
import time
from dataclasses import dataclass
from urllib.error import HTTPError
from urllib.request import Request, urlopen

TTL_SECONDS = 600
MAX_ATTEMPTS = 5
def load_sms_otp_contract(max_retries: int = 3) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    discovery_url = os.environ["INFRAI_SMS_OTP_DISCOVERY_URL"]
    for attempt in range(max_retries + 1):
        request = Request(
            discovery_url,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_retries:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery retry loop ended unexpectedly")


@dataclass
class EmailChallenge:
    digest: str
    expires_at: int
    attempts: int = 0
    consumed: bool = False


def code_digest(secret: bytes, login_id: str, code: str) -> str:
    message = f"{login_id}:{code}".encode()
    return hmac.new(secret, message, hashlib.sha256).hexdigest()


def issue_email_challenge(secret: bytes, login_id: str) -> tuple[str, EmailChallenge]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    challenge = EmailChallenge(
        digest=code_digest(secret, login_id, code),
        expires_at=int(time.time()) + TTL_SECONDS,
    )
    return code, challenge


def verify_email_challenge(
    secret: bytes, login_id: str, submitted_code: str, challenge: EmailChallenge
) -> bool:
    now = int(time.time())
    if challenge.consumed or now >= challenge.expires_at:
        return False
    if challenge.attempts >= MAX_ATTEMPTS:
        return False

    challenge.attempts += 1
    valid = hmac.compare_digest(
        challenge.digest, code_digest(secret, login_id, submitted_code)
    )
    if valid:
        challenge.consumed = True
    return valid


if __name__ == "__main__":
    contract = load_sms_otp_contract()
    print(json.dumps({"method": contract["method"], "path": contract["path"]}, indent=2))
```

Do not log the returned plaintext code. Pass it immediately to the reviewed email template, then discard it. The database record needs the digest, expiry, attempt count, consumed state, login transaction identifier, and normalized destination; it does not need the original code.

Short code does not mean weak controls are acceptable. Add request throttling, destination throttling, IP and account risk checks, and a generic response that does not reveal whether an address exists. The SMS path also needs business-layer geographic fences and per-country spending breakers.

## Fallback is a state transition, not an instant failover

A clean state model is `SMS_PENDING`, `EMAIL_OFFERED`, `EMAIL_PENDING`, then `VERIFIED` or `EXPIRED`. Do not send both challenges at once by default. Offer email after a defined wait or an explicit user action, invalidate superseded challenges, and record which factor completed the login.

There is a hard timing boundary. SMS and email delivery checks are pull-based here; neither namespace supplies webhook events. A worker can poll delivery state and offer fallback, but that is periodic observation, not real-time failover. For an interactive login, a visible "use email instead" action is often clearer than making the user wait for a polling threshold.

Scheduled email adds another constraint: it has `scheduled_at`, but no cancellation route, whereas SMS has cancellation support. That makes scheduled authentication mail a poor fit. Generate fallback codes on demand and expire them in application state. Keep scheduled sending for non-authentication report delivery where the lack of cancellation is acceptable.

For the attached support report, authenticate first, generate second, and send last. Give the generation job an idempotency key derived from the report request so a retry does not create duplicate deliveries. Prompt evaluations should include hostile ticket text, missing fields, and oversized reports; those tests protect the report pipeline, while OTP tests protect entry to it. Different risks. Different harnesses.

## How should this choice be evaluated before release?

Start with correctness, not delivery-provider latency claims. Run the same authentication test matrix against every adapter: expired code, reused code, sixth guess, simultaneous verification, SMS-to-email transition, and a delayed poll arriving after email has already succeeded. Then verify that a vendor swap changes adapter configuration rather than application login logic or template variables.

Measure completion rate by path, time from challenge creation to verification, fallback-selection rate, poll count per login, duplicate-send rate, and lockout rate. Segment operationally relevant delivery data, but avoid storing raw codes or report content in telemetry. A report-generation eval belongs beside these authentication measures: attachment creation success, unsafe prompt leakage, and delivery idempotency all matter before the workflow is ready. Token cost matters for generating the report, yet it should not decide the OTP architecture.

**Choose the hybrid only if the team will own email verification as security code, not as a few lines around an email SDK.** Choose a single managed verification product instead when unified channel orchestration and event-driven status are harder requirements than vendor portability. Choose direct SES, SendGrid, or Postmark integration when an existing email platform and its operational tooling are already the deliberate standard.

The result is slightly asymmetric. Good. The SMS API owns what it can actually manage; Python owns the email code lifecycle; and the reviewed fallback template stays portable. That boundary is easier to test than a generic channel facade whose behavior changes underneath the same method name.

## References

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/how-to-use-the-sendgrid-v3-api/authentication)
- [Postmark API documentation](https://postmarkapp.com/developer/api/overview)
- [NIST Digital Identity Guidelines: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
