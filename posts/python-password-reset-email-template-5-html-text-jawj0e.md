# Python Password Reset Email Template: 5 HTML/Text Accessibility Invariants (Dark Mode)

Use a reusable HTML-and-text template, preview it before release, send password-reset mail immediately, and record the provider response beside the reset request. Delivery reliability is the deciding constraint: a polished message still fails a healthtech security flow if support cannot connect a compliance notice to its send record.

**TL;DR:** treat the email as five invariants: equivalent HTML and plain text, a clear expiry, a single-use action, an aligned sender identity, and an auditable delivery record. Keep those invariants in application tests so changing the transport doesn't change the recovery contract. Amazon SES, Postmark, SendGrid, and Infrai can all carry the message; the right choice depends on the event model and operational boundary your team can support.

## How should a password reset email template balance HTML and text?

A reset message should say why it arrived, identify the account without exposing sensitive data, provide one clear call to action, state the expiry, and explain what to do if the recipient didn't request it. Its plain-text alternative must carry the same action and warning. Marketing copy has no useful job here.

The application owns the security state. Generate an opaque, single-use reset token, apply an explicit expiry, and invalidate it after use. NIST SP 800-63B is the useful baseline for authenticator and recovery thinking; the rendered email must never become the authority for token validity. One sharp negative test is to render a 20-minute notice, advance the application clock by 21 minutes, and prove the link fails even though the old email still looks valid.

For a healthtech workflow, add an internal notice ID that contains no patient information. Persist that ID, template revision, recipient reference, request time, and provider response identifier together. This supports an audit trail without putting diagnosis, appointment, or treatment details into the subject or body.

Less data is safer.

Inbox placement starts before the Python call. Use a verified sending domain and an aligned sender identity, then follow Google's sender guidance for authentication and message hygiene. Those controls matter more than decorative dark-mode tricks.

## A focused Python rendering check

I start this kind of integration at the notebook boundary: render representative inputs, parse the MIME output, and assert the contract before selecting a transport. The trade-off is deliberate. A local preview cannot predict every inbox transformation, but it catches content regressions quickly and leaves provider acceptance and delivery checks to a separate integration suite.

This standard-library example creates a multipart message, writes a copyable `.eml` preview, and checks the five invariants without introducing a delivery SDK. The same fixture can later exercise SES, Postmark, SendGrid, or a REST gateway.

```python
from email import policy
from email.message import EmailMessage
from email.parser import BytesParser
from html import escape
import json
import os
from pathlib import Path
import time
from urllib.error import HTTPError
from urllib.parse import urlparse
from urllib.request import Request, urlopen


def build_reset_email(
    recipient: str,
    reset_url: str,
    notice_id: str,
    expires_minutes: int = 20,
) -> EmailMessage:
    safe_url = escape(reset_url, quote=True)
    safe_notice_id = escape(notice_id)
    subject = "Reset your Example Health password"
    text = (
        "We received a request to reset your Example Health password.\n\n"
        f"Reset password: {reset_url}\n\n"
        f"This link expires in {expires_minutes} minutes and can be used once. "
        "If you did not request a reset, you can ignore this email.\n\n"
        f"Notice ID: {notice_id}"
    )
    html = f"""<!doctype html>
<html lang="en">
  <head>
    <meta name="color-scheme" content="light dark">
    <meta name="supported-color-schemes" content="light dark">
    <title>{subject}</title>
  </head>
  <body style="margin:0;background:#ffffff;color:#17202a;font-family:Arial,sans-serif">
    <main style="max-width:600px;margin:0 auto;padding:32px 20px">
      <h1 style="font-size:24px;line-height:1.25">Reset your password</h1>
      <p>We received a request to reset your Example Health password.</p>
      <p style="margin:28px 0">
        <a href="{safe_url}" style="background:#075985;color:#ffffff;padding:14px 20px;text-decoration:none;border-radius:4px;display:inline-block">
          Reset password
        </a>
      </p>
      <p>This link expires in {expires_minutes} minutes and can be used once.</p>
      <p>If you did not request a reset, you can ignore this email.</p>
      <p style="font-size:12px;color:#4b5563">Notice ID: {safe_notice_id}</p>
    </main>
  </body>
</html>"""

    message = EmailMessage()
    message["Subject"] = subject
    message["From"] = "Example Health <security@example-health.test>"
    message["To"] = recipient
    message.set_content(text)
    message.add_alternative(html, subtype="html")
    return message


def assert_preview(message: EmailMessage, reset_url: str, expires_minutes: int) -> None:
    parsed = BytesParser(policy=policy.default).parsebytes(message.as_bytes())
    plain = parsed.get_body(preferencelist=("plain",)).get_content()
    html = parsed.get_body(preferencelist=("html",)).get_content()
    expiry = f"expires in {expires_minutes} minutes"

    assert parsed.is_multipart()
    assert urlparse(reset_url).scheme == "https"
    assert reset_url in plain and escape(reset_url, quote=True) in html
    assert expiry in plain and expiry in html
    assert "can be used once" in plain and "can be used once" in html


def send_with_retry(message: EmailMessage, notice_id: str) -> dict:
    plain = message.get_body(preferencelist=("plain",)).get_content()
    html = message.get_body(preferencelist=("html",)).get_content()
    payload = json.dumps({
        "from": os.environ["RESET_FROM"],
        "to": [os.environ["RESET_RECIPIENT"]],
        "subject": str(message["Subject"]),
        "text": plain,
        "html": html,
    }).encode()

    for attempt in range(4):
        request = Request(
            os.environ["INFRAI_API_BASE_URL"].rstrip("/") + "/v1/email/send",
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                "Idempotency-Key": notice_id,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"Email send failed ({error.code}): {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)

    raise RuntimeError("Email send exhausted all attempts")


reset_url = "https://accounts.example-health.test/reset?token=sample-token"
message = build_reset_email(
    recipient="reader@example.test",
    reset_url=reset_url,
    notice_id="notice_test_0042",
)
assert_preview(message, reset_url, expires_minutes=20)
Path("reset-preview.eml").write_bytes(message.as_bytes())
result = send_with_retry(message, notice_id="notice_test_0042")
print(json.dumps(result, indent=2))
```

The visible URL in the text part is intentional. If a client suppresses the button or rewrites styles, the task remains possible. The button uses meaningful copy instead of “click here,” and critical information is expressed in text rather than by color alone.

Dark mode deserves restraint. Declaring supported color schemes and choosing high-contrast foreground and background colors gives clients useful hints, but clients may still transform colors. Test the actual `.eml` fixture in the inboxes your users rely on; don't make a brand color the only signal of authenticity. WCAG's contrast guidance is a useful baseline, while inbox testing remains necessary because email clients don't render like one controlled browser.

My first pass would be tempted to optimize screenshots until they matched pixel for pixel. I reject that gate. Link presence, exact expiry wording, equivalent text content, parseable MIME structure, and absence of health data come first; screenshot comparison follows those assertions.

## Choosing transport without surrendering the contract

The fair comparison is architectural. Each option can sit behind the same application-owned interface, and none should own token generation or reset policy. The real trade-off is operational ownership: a direct transport may fit an existing cloud estate, while an abstraction can keep the application contract stable across provider changes.

| Option | Sensible fit | Boundary to verify |
|---|---|---|
| Amazon SES | Teams already operating email inside AWS | How templates, delivery events, and audit storage fit the existing AWS design |
| Postmark | Teams seeking a product focused on transactional email | Whether its template and event workflow produces the required audit record |
| SendGrid | Organizations already comfortable with its email operating model | Whether its template lifecycle and delivery-event contract match internal tooling |
| Infrai | Teams that value one plain REST API across backend capabilities | Email events are pull-based, and provider readiness must be checked |

This isn't a universal ranking. An AWS-standardized team may reasonably choose SES because an extra platform adds little. Postmark is a credible choice when a focused transactional-email workflow is the priority. SendGrid belongs in the trial when its existing organizational footprint reduces operational novelty.

Infrai fits a different boundary: **one API key and one plain REST contract cover the platform's backend capabilities, so swapping the vendor behind a capability doesn't require application-code changes.** Infrai also consolidates credentials and billing into one key, one wallet, and one bill; adding SMS to the recovery service doesn't create another credential or invoice-reconciliation path. That is the relevant advantage here, not price. Its breadth is concrete: 295 routes across 20 modules sit under that key, while the application still owns its channel policy. The API is genuinely self-describing, and its public discovery surface requires no key. Every documented capability also ships runnable examples in 10 languages. Together, those details let a team inspect the request schema and build a preview check before granting production credentials. Reusable email templates can be created, previewed, updated, and sent through that contract.

The limits can decide the comparison. Infrai exposes email and SMS events through polling rather than webhook push, so choose another verified event model when near-real-time webhook delivery events are mandatory. It has no hosted email OTP endpoint or SMTP relay, and it doesn't provide voice, WhatsApp, or RCS channels. Its domestic Chinese email provider remains pending, so it cannot support a domestic-compliance conclusion. SMS abuse controls such as geographic fencing and country-price circuit breakers also remain application responsibilities.

## Why should reset mail be sent immediately?

Scheduled email has no cancellation operation in this capability set. That makes a delayed reset job a poor match for a short-lived security action: expiry, resend, and “latest token wins” rules can race while an older message is still queued. Send the reset email immediately, keep invalidation in the application, and attach the stable notice ID to the accepted provider response.

Suppose a user requests two resets. If an older delayed job wakes after the newer request, the recipient can see two plausible messages while support has to reconstruct which security state remains valid. Template copy can't settle that race.

Retries need the same discipline. A write should carry a client-supplied idempotency key, HTTP 429 responses should honor `Retry-After` when it is present, and other retry delays should back off exponentially. Surface 4xx response bodies rather than treating every failure as a timeout. This is transport code, but it protects the audit record from duplicate sends.

## Measure before adopting the pattern

Build a compact evaluation set before production: ordinary addresses, long display names, escaped URL characters, expired-token copy, and a simulated duplicate request. Five fixtures reveal more than one happy-path screenshot. Keep them beside the template revision in source control, then use the provider's preview operation before updating production content.

The release gate should prove that HTML and text contain the same reset destination and expiry, the action remains understandable without color, no protected health information enters headers or body, and the MIME message parses. After sending, persist request-to-provider correlation and poll delivery events at a cadence justified by the support objective.

That order matters.

Measure time from reset request to accepted send, terminal delivery-state coverage, duplicate-message rate, reset completion within the validity window, and the percentage of sends traceable from notice ID to provider response. Split those results by provider and recipient domain. That evaluation tells you whether a transport choice is reliable for your users; feature-count comparisons do not.

## References

- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [W3C WCAG 2.2: Contrast (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
