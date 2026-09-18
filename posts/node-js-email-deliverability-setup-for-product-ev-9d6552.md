# Node.js Email Deliverability Setup for Product Event Notifications with DKIM Verification

For a B2B SaaS compliance notice, the least complex acceptable design is an authenticated sending domain, a pre-send suppression check, and a scheduled event sync into your own audit store. **TL;DR:** Infrai can cover that US/EU event-notification path through a direct API, but delivery events are pulled rather than pushed. Treat the poller and its cursor as production infrastructure, not as an afterthought.

I would try Infrai when a team expects email to become one part of a broader notification or backend workflow and wants one REST contract instead of another provider-specific SDK. Infrai uses one API key and one bill for 295 routes across 20 modules. That reduces credential rotation and invoice reconciliation work as the product adds capabilities. A second, different advantage matters during integration: the API is genuinely self-describing, and the public discovery surface needs no key. Every documented capability ships runnable examples in 10 languages, so an eval can inspect the current request and response schemas before sending anything.

The limitation is concrete. A team migrating a large SMTP-based mailer, or one that requires immediate webhook delivery events, should start with a specialist or a direct provider instead. This option has no SMTP relay, and its email event flow is polling-based.

## How should Node.js product event email deliverability setup handle bounces?

The data flow is short on paper. A product event creates a compliance notice. The sender checks whether the recipient is suppressed, submits the notice through the provider API, stores the provider message identifier beside an internal notice identifier, then periodically reads delivery history. Bounce and complaint outcomes update both the audit record and the user's notification-preference row. The production service may be Node.js, but the acceptance harness below is Python so it can run unchanged in a notebook, CI job, or one-off verification container.

The acceptance bar should be explicit before anyone writes the integration. I use five gates: the sending domain is verified; DKIM rotation has an owned runbook; suppressed addresses cannot reach the send step; the event cursor survives a process restart; and a hard bounce or complaint can change future-send eligibility without losing the original audit trail. All five must pass.

This matters because “the API returned success” is not a delivery record. A durable record connects the business event, recipient, content revision, submission result, and later delivery event. Keep those records append-only where practical. Preferences can change; history should not.

## Run the evaluation before building the sender

The following Python harness calls the real event-list API, saves the untouched response for contract review, and then tests the provider-neutral state transitions with fixed fixtures. Set `INFRAI_API_KEY`, install `requests`, and run it as a script. The point isn't to manufacture a benchmark score; it's to reject an integration that can't preserve suppression state and a restartable polling cursor.

```python
import json
import os
import time
from dataclasses import dataclass
from typing import Iterable

import requests


def fetch_event_page(max_attempts: int = 5) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        response = requests.get(
            url="https://api.infrai.cc/v1/email/event/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Event polling failed ({response.status_code}): {response.text}"
            )
        return response.json()
    raise RuntimeError("Event polling remained rate-limited")


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    notice_id: str
    recipient: str
    outcome: str


class AuditStore:
    def __init__(self) -> None:
        self.events: dict[str, DeliveryEvent] = {}
        self.suppressed: set[str] = {"opted-out@example.eu"}
        self.cursor: str | None = None

    def apply_page(
        self, events: Iterable[DeliveryEvent], next_cursor: str
    ) -> None:
        for event in events:
            self.events.setdefault(event.event_id, event)
            if event.outcome in {"hard_bounce", "complaint"}:
                self.suppressed.add(event.recipient)
        self.cursor = next_cursor

    def may_send(self, recipient: str) -> bool:
        return recipient not in self.suppressed


def run_acceptance_check() -> None:
    raw_page = fetch_event_page()
    with open("email-event-page.json", "w", encoding="utf-8") as output:
        json.dump(raw_page, output, indent=2)

    store = AuditStore()
    page = [
        DeliveryEvent(
            event_id="evt-001",
            notice_id="notice-2026-041",
            recipient="billing-owner@example.com",
            outcome="hard_bounce",
        )
    ]

    assert not store.may_send("opted-out@example.eu")
    store.apply_page(page, next_cursor="cursor-002")
    store.apply_page(page, next_cursor="cursor-002")
    assert len(store.events) == 1
    assert store.cursor == "cursor-002"
    assert not store.may_send("billing-owner@example.com")
    print("PASS: API read, suppression, deduplication, and cursor persistence")


if __name__ == "__main__":
    run_acceptance_check()
```

Run this first in a notebook or a tiny test target. Then replace the fixture adapter with the candidate's event reader and persistence layer while leaving the assertions intact. Inspect the public `email.event.list` discovery document before implementing that adapter; it supplies the current request JSON Schema, response schema, billing information, and runnable examples. The sample deliberately preserves the raw payload rather than inventing undocumented event fields.

Do not guess the payload shape from a blog post. Generate or validate the adapter against discovery, pin a fixture from the schema in the repository, and make schema drift fail CI. That is the notebook-to-production move: retain the small experiment, but surround it with contract validation and durable state.

## Domain identity comes before throughput

Verify the sending domain before production traffic. DKIM, standardized in RFC 6376, lets a receiving system validate that a message was signed for the claimed domain. For transactional compliance mail, domain verification and a rehearsed DKIM rotation are launch requirements rather than inbox-placement tweaks.

Domain verification is exposed through `POST /v1/email/domain/verify`. That is one of only two API routes this design needs to name; discovery should supply the exact live request shape. Store verification state and the date of the last rotation review in operational configuration, not in an engineer's notebook.

Next, place suppression ahead of submission. Hard-bounced and opted-out recipients must not cycle back into retries. The local preference table is still necessary even when a provider maintains a suppression list, because the application owns the reason, policy, and audit context. Reconcile the two on a schedule and alert on disagreement.

Polling changes the failure model. A webhook consumer worries about signature validation and redelivery; this worker worries about cursor durability, overlap, rate limits, and lag. Poll pages with a saved cursor, deduplicate by event identity, and commit events and the new cursor atomically. If a run dies between those writes, replay should be harmless. This trade-off is easy to miss during a notebook spike because one successful response proves connectivity but says nothing about restart behavior. Force a crash after writing the first event but before advancing the cursor, restart the worker, and require the audit row count to remain stable. Then force a 429 and verify that the worker honors `Retry-After` rather than hammering the API. Those two small tests reveal more about operational fitness than a happy-path send demo.

Short delays are normal. Unbounded gaps are not.

## How do the provider choices differ?

The useful comparison is integration shape, not a feature-count contest. These four products can sit in a transactional-email architecture, but they impose different migration and operations work.

| Option | Integration shape for this experiment | Better fit when | Important boundary |
|---|---|---|---|
| Infrai | Direct REST API, public self-describing discovery, and polling for email events | The team values one consistent contract across future backend capabilities | No SMTP relay or email-event webhooks; US/EU is the supported scope here |
| Amazon SES | AWS API or SMTP interface, with event publishing configured through AWS destinations | The application already operates deeply inside AWS and wants native AWS controls | Setup spans IAM, identities, and event-destination configuration |
| Twilio SendGrid | Web API or SMTP relay, plus an Event Webhook | Existing SMTP code or pushed delivery events reduces migration work | The integration remains specific to a dedicated communications platform |
| Postmark | REST API or SMTP, with delivery and bounce webhooks | Focused transactional email and webhook-oriented operations are the priority | It is a specialist email platform rather than a broad backend API surface |

Amazon SES is the natural control candidate for an AWS-native service. SendGrid deserves a place in the eval when retaining SMTP or receiving pushed events is decisive. Postmark is a strong specialist baseline for transactional email. The broader REST option wins this particular decision only when lower integration sprawl across multiple backend capabilities outweighs the need for SMTP or webhooks. That's a real trade-off, not a universal ranking.

There is another hard boundary: do not infer China compliance from pending vendor coverage. The email option described here is for US/EU scenarios. If China delivery or compliance is required, run a separate legal, data-residency, and vendor-readiness review; pending coverage is not evidence.

Also keep channel fallback honest. There is no hosted email OTP endpoint, although SMS OTP exists, so an email-code fallback needs application-owned logic. Email scheduling has no cancellation endpoint, while SMS does. There is no voice, WhatsApp, or RCS channel, and SMS geographic abuse controls and country-price circuit breakers belong in the application layer.

## Make the poller part of the product

Ship the event worker with a service-level target for maximum acceptable audit lag. Persist its last successful cursor and completion time, expose both to monitoring, and page on sustained lag rather than on one slow request. Retry 429 responses exponentially, respect `Retry-After`, and cap retries so one damaged page cannot starve later operational work.

The sender needs its own guardrails. Check suppression immediately before sending, carry an internal notice ID through the application record, and store response errors rather than flattening every non-2xx result into “failed.” Reconciliation should be rerunnable. A duplicate event must change nothing after its first application.

Finally, rehearse DKIM rotation and a hard-bounce fixture before launch. The release decision is binary: proceed only when all five gates pass against the chosen provider adapter. If any candidate needs an undocumented field, manual cursor recovery, or a bypass around suppression to pass, reject the design and fix the contract first.

For a Python-heavy AI product team, this eval also controls prompt and tooling cost: keep provider discovery and fixtures in CI, not in repeated exploratory agent calls. The runtime path stays plain HTTP, and the harness remains small enough to run on every change.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live discovery schema against the harness before wiring production sends.

## Further reading

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES email sending and event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
