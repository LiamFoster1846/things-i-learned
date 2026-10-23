# Media Import Capture: Implementing 8-Field Python FastAPI NodeJS Error Correlation

Short answer: for error tracking in a mixed Python FastAPI and NodeJS stack, standardize one eight-field event at every microservice boundary, preserve the incoming W3C trace ID, and send events to a small capture endpoint that validates before storage. For scheduled media imports, the deciding constraint is incident reconstruction: an alert saying “zero stories imported” is useful only if an engineer can connect the scheduler run, fetch failure, parser error, and final empty result without guessing.

The tempting first pass is to grep Python and Node.js logs by timestamp. It fails when clocks differ, retries overlap, or one service turns an exception into a successful-but-empty response. The stronger design treats the import run ID as the business correlation key and the trace ID as the request-path key. Keep both. That gives an eval harness something deterministic to assert, while keeping verbose prompt or document content out of the event stream.

## How should Python FastAPI and NodeJS capture mixed-stack errors?

A trace ID describes a distributed execution path. A scheduled import is a business operation that may create several paths: the scheduler dispatches a job, a worker retries later, and a downstream parser may process batches independently. If every retry receives a new trace, searching only by trace ID fragments the story. If code reuses one trace forever, it misrepresents separate executions and creates an unwieldy correlation group.

Use two identifiers with different lifetimes. `import_run_id` stays stable for one scheduled run, including its retries. `trace_id` follows the W3C Trace Context format and identifies one distributed request path. The W3C `traceparent` field carries a version, trace ID, parent ID, and trace flags; the trace ID is 16 bytes represented as 32 lowercase hexadecimal characters. Generate a run ID at the scheduler, accept or create trace context at each ingress, and propagate both explicitly.

This distinction catches a media-specific blind spot. A feed can return HTTP 200, the parser can reject every item, and the publishing stage can receive an empty list. There may be no uncaught exception at all. Emit a terminal `no_results` event when a completed run produces zero accepted records, then alert on that event or on the absence of a successful terminal event by the schedule deadline.

Silence matters.

Picture the reconstruction before choosing fields. At 08:00, the scheduler creates run `42d8b70e-69c6-4bb6-a62e-31d4dc4ddf31` and dispatches a fetch. That request gets one trace ID. The feed responds, twelve records reach parsing, and all twelve are rejected because the document type is unsupported. A retry at 08:05 belongs to the same import run but starts another request path, so it may have another trace ID. The second attempt sees the same twelve records and reaches the same terminal outcome. Grouping on service name would scatter the evidence; grouping only on the first trace would hide the retry; grouping on the run ID restores the timeline, while each trace still explains calls inside one attempt. This is the concrete query the schema must support. It also explains why a generic exception message is weak evidence: “parse failed” does not say whether the importer saw zero records, rejected twelve, or lost the result after parsing.

## Define the eight-field reconstruction contract

The event should answer four questions quickly: which run failed, where did it fail, what kind of failure occurred, and when was it observed? The following contract is deliberately small enough to implement in both runtimes without coupling application code to a storage engine.

| Field | Type | Reconstruction purpose |
|---|---|---|
| `occurred_at` | RFC 3339 timestamp string | Orders evidence across services |
| `import_run_id` | UUID string | Groups retries and stages for one scheduled run |
| `trace_id` | 32-character lowercase hex string | Connects one distributed request path |
| `service` | bounded string | Locates the emitting component |
| `stage` | bounded enum-like string | Shows scheduler, fetch, parse, or publish position |
| `outcome` | bounded enum-like string | Separates error, retry, no-results, and success states |
| `error_type` | bounded string or null | Groups stable failure classes without stack text |
| `detail` | object | Holds allowlisted diagnostic attributes |

Do not make `detail` a dump of local variables. OWASP advises excluding or masking access tokens, passwords, sensitive personal data, connection strings, and other secrets from logs. For this pipeline, useful allowlisted attributes include `feed_host`, `http_status`, `attempt`, `items_seen`, and `items_accepted`. Article bodies, authorization headers, raw prompts, and model responses do not belong here. That boundary controls exposure and keeps token-heavy payloads from inflating ingestion and retention.

Cardinality is a design choice, too. Keep `service`, `stage`, `outcome`, and `error_type` bounded so grouping remains meaningful. IDs are high-cardinality by nature and should be used for lookup, not dashboard grouping. Human-readable messages may change; stable fields should carry the query.

The trade-off is extra application discipline. Every producer must preserve two identifiers and agree on outcome semantics. This design is a poor fit for a single-process importer with one log stream and no retries, where structured local logs may already reconstruct the whole run. It is also unsuitable when the team cannot operate an authenticated, durable ingestion path; in that case, use an established telemetry collector that accepts a standard protocol and keep this event contract at the application boundary. Owning a custom endpoint means owning backpressure, schema evolution, access control, retention, and dropped-event monitoring. The eight fields reduce ambiguity, but they do not remove that operational load.

## Implement validation and capture

Start in a notebook or a focused test with representative events: fetch timeout, malformed feed, partial parse, and zero accepted items. Then move the same schema into the capture service. The example below is a complete Python module for validation and an HTTP endpoint; another runtime should emit the same JSON shape and validate it against an equivalent JSON Schema or typed model.

```python
from datetime import datetime, timezone
from typing import Any, Literal
from uuid import UUID

from fastapi import FastAPI, Response, status
from pydantic import BaseModel, ConfigDict, Field, field_validator


Stage = Literal["scheduler", "fetch", "parse", "publish"]
Outcome = Literal["error", "retry", "no_results", "success"]


class ErrorEvent(BaseModel):
    model_config = ConfigDict(extra="forbid")

    occurred_at: datetime
    import_run_id: UUID
    trace_id: str = Field(pattern=r"^[0-9a-f]{32}$")
    service: str = Field(min_length=1, max_length=64)
    stage: Stage
    outcome: Outcome
    error_type: str | None = Field(default=None, max_length=96)
    detail: dict[str, Any] = Field(default_factory=dict)

    @field_validator("occurred_at")
    @classmethod
    def require_timezone(cls, value: datetime) -> datetime:
        if value.tzinfo is None or value.utcoffset() is None:
            raise ValueError("occurred_at must include a UTC offset")
        return value


app = FastAPI()


@app.post("/events", status_code=status.HTTP_202_ACCEPTED)
async def capture_event(event: ErrorEvent, response: Response) -> None:
    normalized = event.model_copy(
        update={"occurred_at": event.occurred_at.astimezone(timezone.utc)}
    )
    await append_event(normalized.model_dump(mode="json"))
    response.headers["Cache-Control"] = "no-store"


async def append_event(event: dict[str, Any]) -> None:
    # Replace with a bounded queue or durable writer owned by the deployment.
    print(event)
```

The endpoint returns `202 Accepted`, which means the request was accepted for processing; it does not promise durable storage by itself. The production implementation therefore needs a clear acknowledgment boundary. Either append to durable storage before responding, or enqueue to a durable broker whose successful write defines acceptance. A process-local background task can lose accepted events during a crash, exactly when reconstruction evidence is most valuable.

Keep the ingestion path boring. Apply request-size limits at the server or proxy, authenticate emitters, rate-limit abusive clients, and reject unknown fields. Normalize timestamps to UTC while retaining the original instant. Storage can be an analytical database, an object-backed event stream, or another append-oriented system; the contract should not depend on its query syntax.

## Test the incident, not just the endpoint

An endpoint test proves very little.

The useful eval runs a known import fixture through the pipeline and checks the evidence produced at each decision boundary. This is the same discipline that makes an AI feature shippable: define the expected outcome before tuning the mechanism.

For one malformed-feed fixture, assert that all emitted events share one `import_run_id`; each event has a valid trace ID; a parse-stage error has a stable `error_type`; and the run finishes with `no_results`, not `success`. For a partial fixture, assert `items_seen` and `items_accepted` are present, while article text is absent. Add a retry case where the run ID stays fixed but a new request path may carry a new trace ID.

```python
from uuid import uuid4

from fastapi.testclient import TestClient


def test_capture_rejects_unbounded_payload_fields() -> None:
    client = TestClient(app)
    event = {
        "occurred_at": "2026-10-17T08:15:00Z",
        "import_run_id": str(uuid4()),
        "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
        "service": "feed-parser",
        "stage": "parse",
        "outcome": "no_results",
        "error_type": "unsupported_document",
        "detail": {"items_seen": 12, "items_accepted": 0},
        "raw_article_body": "must not be captured",
    }

    response = client.post("/events", json=event)

    assert response.status_code == 422
```

Also test propagation. Given an incoming valid `traceparent`, the downstream request should carry the same trace ID and a new parent ID. Given no valid context, ingress should create new context. Do not invent trace identifiers by hashing user data; use a conforming tracing library in each runtime, because sampling flags and parent relationships are part of the protocol as well.

One check is non-negotiable: kill the capture process after acknowledgment and verify that the accepted event remains available.

The alert eval needs time semantics. Run the checker against a schedule fixture and verify it fires only after the defined completion window, distinguishes “job never started” from “job completed with no results,” and links to a query keyed by `import_run_id`. Avoid a magic global timeout: a five-minute feed and a two-hour archive import need separate expectations.

## Reconstruct first, then tune volume

During an incident, begin with the missing schedule slot and find its run ID. Order all matching events by `occurred_at`, then split by trace ID to see individual attempts. The first terminal failure is often more informative than the last retry message, while the counts in `detail` reveal whether work disappeared during fetch, parse, or publish.

Measure before copying this design unchanged. Track accepted versus rejected capture requests, ingestion delay, missing terminal events, events per import run, and bytes per event. Those measurements expose schema drift and telemetry loss. They also put prompt-cost awareness in the right place: record model name, token counts, or evaluation outcome only when an import stage actually invokes a model and the values are available from the runtime; never attach full prompts by default.

Retention should follow reconstruction needs rather than habit. Keep enough detail to cover the incident-review window, restrict access, and verify deletion behavior. Analytical stores such as ClickHouse can query high-volume event data, but storage choice does not repair an inconsistent schema or missing propagation. The decisive test remains simple: can an engineer explain why a scheduled import produced nothing, across every attempt, from bounded and non-sensitive evidence?

**Choose the smallest contract that passes that reconstruction eval.** Add fields only when a failed investigation demonstrates a specific unanswered question. This keeps Python and Node.js emitters aligned, protects the capture endpoint from arbitrary payload growth, and leaves the backend replaceable.

## References

- W3C, “Trace Context”: https://www.w3.org/TR/trace-context/
- OWASP, “Logging Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- MDN, “202 Accepted”: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/202
- ClickHouse documentation: https://clickhouse.com/docs
