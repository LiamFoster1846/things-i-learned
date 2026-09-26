# FastAPI SMS Provider Experiments: Compare Startup Alerts, Templates, and Signatures

A startup choosing an SMS provider for US and EU support alerts should run a delivery experiment, not crown a winner from a feature grid. Put Twilio, Plivo, Telnyx, and Sinch behind the same tiny adapter; replay the same contact-form events; then compare accepted, delivered, duplicated, delayed, and unverified-callback outcomes by destination. Templates and signatures matter, but only as parts of that reliability test.

**Short answer:** choose the candidate that meets your measured delivery objective in both regions while preserving consent, opt-out handling, sender registration, and auditable webhook verification. Keep the queue as the source of truth. A provider's API acceptance is not proof that a shopper's phone received the alert.

This experiment has one concrete job: route an e-commerce contact form to the right support queue, then alert the on-call team by SMS. The tempting first pass sends during the FastAPI request and treats a successful API response as completion. It is compact. It also ties form latency and durability to a remote dependency, and it gives retries nowhere trustworthy to live.

The chosen design commits the contact event first, enqueues an alert with an idempotency key, and lets a worker call a replaceable gateway. Delivery callbacks update state only after signature verification. That extra state is worthwhile because delivery reliability is the primary decision axis here.

Acceptance is provisional.

## How should a startup compare an SMS provider for support alerts?

Start with a frozen corpus, because changing messages while changing providers ruins the comparison. Use at least three template shapes: a short ASCII alert, a long alert near a segment boundary, and a message containing a non-GSM character such as an emoji. SMS length is not one universal character count: GSM-7 messages and UCS-2 messages have different single-message and concatenated limits, and concatenation consumes header space. That can change segment count and the chance that a long alert arrives in pieces.

Use synthetic contact records, never production phone numbers or customer text. Each event should carry a region, queue, template version, consent-state fixture, and stable event ID. Send identical batches through each candidate at controlled rates. Repeat at different times, but do not pretend a small trial establishes universal carrier performance; it establishes performance for your destinations, message mix, sender setup, and test window.

The scorecard should record these fields for every attempt:

| Signal | Why it matters | Failure-injection check |
| --- | --- | --- |
| API acceptance | Separates request failures from downstream outcomes | Force timeouts before and after acceptance |
| Verified status callback | Prevents an unauthenticated request from changing delivery state | Alter the body and signature |
| Final delivery state | Measures the outcome the alerting workflow needs | Exercise invalid and unreachable test destinations |
| End-to-end latency | Exposes queueing and callback delay | Add worker backpressure |
| Segment count | Finds encoding-driven expansion | Compare ASCII with emoji fixtures |
| Duplicate count | Tests retry safety | Deliver the same queue item twice |

Do not combine these into one magic score too early. A provider with a lower median latency can still be a poor fit if its tail misses the escalation window. Decide the service objective first, including the percentile and time window, then evaluate the observations against it.

This method has a real limitation: it is not suitable for a team that cannot maintain a durable queue, callback verification, and regional test numbers. In that case, choose a simpler managed workflow that exposes delivery evidence and suppression controls, then accept the trade-off of less portability. The experiment also cannot predict performance in countries, carrier paths, or sender configurations it did not test.

## Build the seam before comparing candidates

The application should know about an alert command, not four vendor SDK object models. This Python sketch keeps the notebook-to-production path honest: the experiment harness and worker call the same protocol, while every attempt is tied to one event and one template version.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Alert:
    event_id: str
    destination: str
    body: str
    region: str
    template_version: str


@dataclass(frozen=True)
class Submission:
    provider_message_id: str
    accepted: bool


class SmsGateway(Protocol):
    def send(self, alert: Alert, idempotency_key: str) -> Submission: ...


def dispatch(alert: Alert, gateway: SmsGateway) -> Submission:
    return gateway.send(alert, idempotency_key=f"contact:{alert.event_id}")
```

The idempotency key is an application invariant, not an assumption that every API interprets retries identically. Persist an attempt row before the network call. On an ambiguous timeout, reconcile recorded provider IDs and callbacks before sending again. If the gateway cannot prove whether it accepted the first request, the worker needs an explicit policy: wait for reconciliation, escalate through another channel, or risk a duplicate. Hiding that choice inside a generic retry decorator makes the experiment look cleaner than production will be.

Keep the FastAPI request short. Validate and store the contact event, select the support queue with a deterministic rule, enqueue the alert, and return. An order-status form can route by market and issue type; free-text model output can suggest a queue, but a bounded rule should decide where safety or service objectives matter. An eval set of labeled forms belongs in CI so a prompt edit cannot quietly redirect billing complaints to general support. The model's confidence is not a delivery signal.

There is a useful cost connection here, though price is not the thesis. A verbose model-generated alert may cross an encoding or segmentation boundary. Render a compact, versioned template after classification instead of sending raw model prose. That controls prompt spillover, makes messages reviewable, and lets the harness assert the exact body sent.

## Templates, signatures, and compliance belong in the test

“Easy templates” is too vague to score. Define the operations the team needs: version a template, preview the exact encoded body, reject missing variables, estimate segments, stage a rollout, and identify the version from an attempt record. A repository-backed renderer can satisfy those needs. A hosted editor may also satisfy them. The experiment should judge the workflow and output, not the screenshot.

Webhook signatures need the same treatment. Each adapter must verify the provider's documented signing scheme against the raw request representation it specifies, reject malformed callbacks, and bind the callback to a known message ID. Do not copy one vendor's verification algorithm into a supposedly generic helper. Twilio, Plivo, Telnyx, and Sinch remain separate adapters because their callback contracts and verification instructions are provider-specific. Test each adapter with official or locally captured test-mode fixtures, including a modified body and a missing signature.

Compliance is configuration plus evidence, not a checkbox named “US/EU.” Requirements depend on destination, use case, sender type, and current rules. The selection exercise should ask every candidate the same operational questions: Can the startup configure the required sender identity for its actual traffic? Where are consent and opt-out events recorded? Can suppression be enforced before enqueueing? Is there an exportable audit trail tying message purpose, template version, recipient state, and delivery attempt together? Legal counsel and the provider's current country guidance must resolve jurisdiction-specific obligations before launch.

This produces an objective comparison without inventing a universal ranking. Run all four candidates through the same matrix and record pass, fail, or not tested.

A missing test is not a pass.

## Failure injection is the useful demo

Happy-path notebooks make every JSON API look easy. The experiment becomes informative when the worker is killed after a remote acceptance but before its local commit. Run that case repeatedly. Also replay callbacks out of order, deliver one callback twice, delay callbacks beyond the escalation window, and return malformed payloads.

State transitions should be monotonic where the provider contract permits it, and callback processing should be idempotent. Store the raw callback separately from normalized state so an adapter bug can be audited without rewriting history. Redact message bodies and phone numbers from routine logs; correlate with internal event and attempt IDs instead.

One trap deserves emphasis: failover can increase duplicates. Sending through candidate B immediately after candidate A times out may produce two alerts if A accepted the request before the connection broke. For a support notification, a delayed but reconciled retry may be preferable to waking two agents with two apparently independent incidents. Choose deliberately.

Retries lie unless state survives them.

The same harness can test regional behavior without hard-coding a winner. Parameterize destination fixtures, sender configuration, template, and gateway; export raw observations; and calculate metrics afterward. Keep the evaluator boring and deterministic.

```python
from dataclasses import dataclass
from statistics import median


@dataclass(frozen=True)
class Observation:
    accepted: bool
    delivered: bool
    duplicate: bool
    callback_verified: bool
    latency_seconds: float | None


def summarize(rows: list[Observation]) -> dict[str, float | int]:
    delivered_latencies = [
        row.latency_seconds
        for row in rows
        if row.delivered and row.latency_seconds is not None
    ]
    return {
        "attempts": len(rows),
        "delivered": sum(row.delivered for row in rows),
        "duplicates": sum(row.duplicate for row in rows),
        "unverified_callbacks": sum(not row.callback_verified for row in rows),
        "median_delivery_seconds": (
            median(delivered_latencies) if delivered_latencies else -1.0
        ),
    }
```

The `-1.0` sentinel is intentionally visible, but production reporting should model “no delivered samples” as missing rather than charting it as fast delivery. Small details like that prevent a blank cohort from winning a dashboard comparison.

## Decide from evidence, then keep measuring

Before copying this design, measure the real message corpus: encoding distribution, segments per alert, queue delay, accepted-to-delivered latency by percentile, verified-callback rate, duplicate rate, final unknown rate, and escalation outcomes. Slice by destination region and sender configuration. Track template and adapter versions so a regression has a boundary.

The selection decision should be a written acceptance test. It might require zero unauthenticated state changes, deterministic suppression before send, bounded duplicate behavior under the kill-after-acceptance test, and a stated delivery objective for each launch region. Set the actual thresholds from business impact and a representative trial; no public comparison can supply them honestly.

This approach may select different gateways as traffic and destinations change. Good. The durable result is the experiment, the audit trail, and a queue that does not confuse API acceptance with delivery. Those are what keep a contact-form alert reliable after the demo ends.

## Further reading

- [Twilio: SMS character limits and segmentation (GSM-7/UCS-2)](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
