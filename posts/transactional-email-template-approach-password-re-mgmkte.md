# Transactional Email Template Approach: Password Reset Localization Meets Settled Receipts

TL;DR: Keep transactional email templates in the application repository, render them from typed event data, and preview every supported locale before deployment. Send an edtech order receipt only after the payment system records a settled transition, using the payment event ID as the idempotency key. Treat password recovery as a separate message family with the same rendering pipeline but a stricter data boundary: the template receives an expiring, single-use URL, never a password or an account-existence signal.

This decision favors delivery reliability over visual editing convenience. A template that looks polished but can be emitted twice, translated with a missing variable, or triggered before settlement is the wrong template approach. The useful unit to test is not an HTML file. It is the combination of event, locale, subject, text alternative, HTML output, and send policy.

## What should a transactional email template approach share with password reset?

The application should own content versions, locale selection, variable validation, escaping, and preview fixtures. A delivery adapter should own the final handoff and return a provider-neutral result such as an accepted message identifier or a categorized error. The payment service remains the authority on whether an order has settled. Those boundaries prevent a copy edit from changing business timing and prevent a delivery retry from charging or fulfilling an order again.

Keep that line sharp.

For an edtech checkout, the flow is short in prose: the payment integration verifies an incoming event, the order service applies the state transition once, and an outbox record captures `order.receipt_requested` in the same durable transaction. A worker later renders the student's locale and calls the delivery adapter. If that call times out, the worker retries the outbox item; it does not replay payment settlement.

This separation matters even for a small notebook-born feature. The first version often renders and sends inside a web request because it is easy to inspect. Production introduces delayed callbacks, duplicate events, process restarts, and translation changes. Moving the durable intent into an outbox gives the eval harness a stable input and keeps network uncertainty away from the order state machine.

## Render one event into inspectable artifacts

The following focused example uses only the Python standard library. It validates the event shape, formats money from minor units, escapes all untrusted values, renders both MIME alternatives, and writes a browser preview. Its locale catalog is intentionally tiny; production catalogs should come from the team's reviewed localization workflow rather than being expanded inline.

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from html import escape
from pathlib import Path
from string import Template


COPY = {
    "en-US": {
        "subject": "Receipt for order ${order_id}",
        "heading": "Payment received",
        "intro": "Your enrollment payment has settled.",
        "total": "Total",
        "open_order": "View order",
    },
    "es-ES": {
        "subject": "Recibo del pedido ${order_id}",
        "heading": "Pago recibido",
        "intro": "El pago de tu matricula se ha completado.",
        "total": "Total",
        "open_order": "Ver pedido",
    },
}


@dataclass(frozen=True)
class SettledOrder:
    order_id: str
    payment_event_id: str
    student_name: str
    locale: str
    currency: str
    total_minor: int
    order_url: str


@dataclass(frozen=True)
class RenderedEmail:
    subject: str
    text: str
    html: str


def money(total_minor: int, currency: str) -> str:
    if total_minor < 0 or len(currency) != 3:
        raise ValueError("invalid monetary value")
    amount = Decimal(total_minor) / Decimal(100)
    return f"{currency.upper()} {amount:.2f}"


def render_receipt(order: SettledOrder) -> RenderedEmail:
    if order.locale not in COPY:
        raise ValueError(f"unsupported locale: {order.locale}")
    if not order.order_url.startswith("https://"):
        raise ValueError("order_url must use HTTPS")

    copy = COPY[order.locale]
    safe_name = escape(order.student_name)
    safe_order_id = escape(order.order_id)
    safe_url = escape(order.order_url, quote=True)
    safe_total = escape(money(order.total_minor, order.currency))

    subject = Template(copy["subject"]).substitute(order_id=order.order_id)
    text = (
        f'{copy["heading"]}\n\n{copy["intro"]}\n'
        f'{copy["total"]}: {safe_total}\n{copy["open_order"]}: {order.order_url}'
    )
    html = f"""<!doctype html>
<html lang="{escape(order.locale)}">
<head><meta charset="utf-8"><title>{escape(subject)}</title></head>
<body>
  <main>
    <h1>{escape(copy["heading"])}</h1>
    <p>{safe_name}, {escape(copy["intro"])}</p>
    <p><strong>{escape(copy["total"])}:</strong> {safe_total}</p>
    <p><a href="{safe_url}">{escape(copy["open_order"])}</a></p>
    <p>Order {safe_order_id}</p>
  </main>
</body>
</html>"""
    return RenderedEmail(subject=subject, text=text, html=html)


def write_preview(message: RenderedEmail, destination: Path) -> None:
    destination.write_text(message.html, encoding="utf-8")


sample = SettledOrder(
    order_id="EDU-10482",
    payment_event_id="evt_7f62a1",
    student_name="Amina & Family",
    locale="en-US",
    currency="usd",
    total_minor=12900,
    order_url="https://school.example/orders/EDU-10482",
)
preview = render_receipt(sample)
write_preview(preview, Path("receipt-preview.html"))
```

The sample produces a deterministic artifact for review without making a network call. That is deliberate. A preview command should be safe to run in a pull request or notebook, and the exact rendered strings can become golden files. I use one fixture per locale plus adversarial fixtures containing `&`, quotes, long names, zero-value orders, and the largest supported amount. The trade-off is repository weight; reviewed snapshots are worth it for high-risk messages, while low-risk copy can use structural assertions.

Preview first.

One caveat is important: `USD 129.00` is deterministic, but it is not a complete locale-aware currency formatter. A real localization library backed by reviewed locale data should handle symbol placement, grouping, plural rules, and scripts. The example keeps that concern visible instead of pretending that swapping dictionary strings finishes internationalization.

## Why can a correct preview still produce an unreliable receipt?

Preview tests answer “what will this event render?” They do not answer “should this event exist?” or “was it already delivered?” Reliability therefore needs checks on both sides of rendering. Before rendering, require the order state to be settled and persist a unique key such as `(message_kind, payment_event_id)`. A database uniqueness constraint is stronger than an in-memory “already sent” flag because concurrent workers and restarts are normal operating conditions. After handoff, retain the template version, locale, event key, attempt count, and delivery system's message identifier. Do not log the full HTML, recovery token, or complete recipient address merely to make dashboards convenient. Acceptance is not delivery: an API can accept a message while a downstream mailbox rejects, defers, or filters it, so model those as separate states, consume authenticated delivery events where available, and alert on age and terminal failure by message class. A receipt delayed for ten minutes has different consequences from a weekly digest; the queue policy should know that. Retries need classification as well. Retry timeouts, connection failures, and explicit temporary responses with bounded exponential backoff and jitter. Route permanent address or policy failures to review instead of hammering the same destination. Cap attempts, preserve the original idempotency key, and expose queue age.

Fast retries prove nothing.

For password recovery, keep the rendering pattern but change the contract. OWASP recommends a consistent response for existing and nonexistent accounts, cryptographically random tokens, secure storage, single use, and expiration. The reset URL should be constructed from trusted configuration rather than a request `Host` header. Never place the token in logs or analytics parameters. A text alternative is still required, and generic wording should avoid revealing whether an account exists.

## Evaluate content before paying for a send

The cheapest test message is the one never submitted. A compact offline harness can render a matrix of locales and fixtures, parse the HTML, and assert invariants: one document language, one visible order identifier for receipts, an HTTPS action URL, no unresolved `${...}` placeholders, and a nonempty text alternative. Snapshot diffs then make copy changes reviewable.

Add two negative cases early: an unsupported locale must fail closed, and an event whose state is merely authorized must not create a receipt intent. Those tests catch more serious failures than pixel comparison. For credential recovery, assert that raw tokens never appear in structured logs and that the rendered link comes only from the dedicated recovery-link field.

This is where prompt-cost awareness helps even though no model belongs in the send path. If an AI system drafts translations or classifies support replies, evaluate its output offline and require human approval before content enters the catalog. Do not invoke a model per receipt. Deterministic rendering is faster to test, has a fixed computational cost, and does not allow wording to drift between two students who completed the same transaction.

Visual inspection still matters because email clients implement a constrained and uneven subset of web layout. Keep receipt HTML conservative, include meaningful link text, declare the document language, maintain readable contrast, and avoid communicating payment status through color alone. Browser preview is the first gate, not proof of identical rendering in every mailbox.

## Put the pipeline into production deliberately

Start deployment with a shadow render: consume representative, redacted events and generate artifacts without handing them to a delivery system. Review every locale, then enable a small internal recipient set. Once real delivery is enabled, watch outbox age, attempts per message, permanent failure rate, and the time between settlement and accepted handoff. These measures identify different breakpoints; collapsing them into a single “email success” counter hides useful evidence.

The operational checklist is prose because the order matters. First, verify event authenticity and apply settlement idempotently. Next, commit the order transition and outbox intent together. Let a worker validate the event contract, select an explicitly supported locale, render both alternatives, and submit through a narrow adapter with the stable event key. Record metadata without sensitive content, reconcile authenticated delivery events, and expire retained details under the organization's data policy. Finally, rehearse a delivery outage: queued receipts should wait without blocking payment settlement, duplicating enrollment, or issuing premature recovery messages.

Keep content changes deployable independently only if version pinning remains explicit. Otherwise a retry may render different copy from the first attempt. Pinning a catalog version in the outbox costs a little storage and release plumbing, but it makes an incident explainable and a replay reproducible.

That is the durable approach: application-owned, locally previewable content attached to a transactional outbox and a replaceable delivery boundary. The template is small. The reliability work surrounds it.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- W3C Web Content Accessibility Guidelines 2.2: https://www.w3.org/TR/WCAG22/
- W3C Internationalization, Language tags in HTML and XML: https://www.w3.org/International/articles/language-tags/
- Python documentation, `html.escape`: https://docs.python.org/3/library/html.html#html.escape
- Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
- RFC 5322, Internet Message Format: https://www.rfc-editor.org/rfc/rfc5322
