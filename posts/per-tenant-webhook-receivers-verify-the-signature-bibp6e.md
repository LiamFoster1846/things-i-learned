# Per-tenant webhook receivers — verify the signature, enqueue raw body, acknowledge fast

Use one rule for webhook intake and let it settle every other argument: verify the signature over the raw request body, put those exact bytes on a queue, and acknowledge before any real work starts. Parsing, grading, the database write, the notification email — all of that belongs to a consumer nobody is waiting on.

Three steps. Nothing else in the request path.

I build RAG and auto-grading features for an edtech product, so the intake endpoint sits in front of about 40 school districts, and each district is a tenant with its own signing secret and its own scoped API key. That shape is why this article exists at all. The primary question for a receiver like this isn't throughput, it's blast radius: if one district's integration leaks a credential, I want to revoke exactly that credential, not a shared secret that 39 other districts are also signing with. A single shared webhook secret is convenient for about a week — then somebody pastes it into a support ticket or a public runbook, and the only honest remediation is rotating it everywhere at once, at 2am, while every district's events pile up in their sender's retry queue.

## Should I verify the signature before I enqueue the raw body?

Yes, and the ordering isn't a matter of taste. An unverified payload should never take up a queue slot, a worker, or a row in your events table; verification is a few microseconds of HMAC over a few kilobytes, and it's the only thing standing between your queue and anyone who can guess the URL.

The part that catches people is the word *raw*.

A signature covers bytes, not meaning. Once a JSON parser has run you're holding an object, and re-serializing that object gives you *a* representation rather than *the* one that was signed — key order, whitespace, `\uXXXX` escapes and float formatting all drift, and the HMAC drifts with them. In Express that makes `express.json()` on a webhook route an easy trap, because by the time your handler runs the original bytes are gone. Mount `express.raw({ type: 'application/json' })` on the webhook path only, or pass a `verify` callback into `express.json()` that stashes the buffer on `req.rawBody` before parsing happens. The Python equivalents are `request.get_data()` in Flask and `await request.body()` in FastAPI. Same idea in any runtime: capture bytes first, parse second, and never let a framework convenience sit between the two.

Two more things belong in the same check. Compare digests with a constant-time function, not `==`. And if the sender signs a timestamp alongside the payload — most do — reject anything outside a five-minute window, so a captured delivery can't be replayed at you a month later.

## The intake path, end to end

The flow is short enough to hold in your head. A district's system POSTs to `/hooks/<tenant_id>`; the receiver looks up that tenant's secret, recomputes the HMAC over the raw bytes, base64-encodes the body into a queue message keyed by the sender's event id, and returns 204. A separate worker pulls from that queue and does the slow part — the grading run, the roster sync, the model call — with nobody holding a socket open. Registration is where the secret comes from in the first place: you hand the sender your endpoint URL and the list of events you want, and it hands back the signing secret that makes any verification possible later.

Here's the receiver, in the language I actually ship in. The Node.js version is the same three steps with `express.raw` instead of `get_data()`.

```python
import base64, hashlib, hmac, os, time
import requests
from flask import Flask, request

app = Flask(__name__)

API = os.environ["INFRAI_API_BASE"]        # API host root, from the environment
KEY = os.environ["INFRAI_API_KEY"]         # project key, ifr_... — never a literal
QUEUE = "webhook-intake"
SIG_HEADER = "X-Signature-256"             # whatever the sender registered with
MAX_SKEW_SECONDS = 300


def tenant_secret(tenant_id: str) -> bytes:
    """One secret per district, so revoking one never touches the other 39."""
    return os.environ[f"WEBHOOK_SECRET_{tenant_id.upper()}"].encode()


@app.post("/hooks/<tenant_id>")
def receive(tenant_id: str):
    raw = request.get_data()                       # bytes, before any JSON parsing
    ts = request.headers.get("X-Timestamp", "0")
    if abs(time.time() - float(ts)) > MAX_SKEW_SECONDS:
        return {"error": "stale timestamp"}, 400

    signed = f"{ts}.".encode() + raw
    expected = hmac.new(tenant_secret(tenant_id), signed, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(request.headers.get(SIG_HEADER, ""), expected):
        return {"error": "bad signature"}, 401

    event_id = request.headers.get("X-Event-Id") or hashlib.sha256(signed).hexdigest()
    publish(tenant_id, event_id, raw)
    return "", 204


def publish(tenant_id: str, event_id: str, raw: bytes) -> None:
    message = {
        "queue": QUEUE,
        "payload": {
            "tenant_id": tenant_id,
            "event_id": event_id,
            "body_b64": base64.b64encode(raw).decode(),
        },
        "delay_seconds": 0,
    }
    for attempt in range(4):
        r = requests.post(
            f"{API}/v1/queue/publish",
            headers={
                "Authorization": f"Bearer {KEY}",
                "Idempotency-Key": event_id,       # a retried publish lands once
            },
            json=message,
            timeout=3,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        r.raise_for_status()                       # a 4xx body carries the reason
        return
    raise RuntimeError("publish retries exhausted")
```

Two details in there earn their keep. The `Idempotency-Key` header means a network hiccup during publish doesn't duplicate the event, and base64 keeps the bytes intact so a consumer can re-verify the signature days later if an auditor asks what a district actually sent. Standard queues are at-least-once, so the consumer has to be safe to run twice on the same `event_id` — mine upserts on that id before it touches anything expensive.

## Where the webhook platforms actually differ

Once the receiver exists, the real decision is how much of retries, replay, dead-letter handling and key lifecycle you want to own.

| Option | What you get | How you integrate | Where it stops |
| --- | --- | --- | --- |
| Hand-rolled receiver + your own queue | Total control of the raw bytes and the ack budget | Your framework, your infrastructure | You own retries, DLQs, replay and secret rotation |
| Svix | Signed outbound delivery plus verification libraries | SDKs in most languages | Built for sending your events; inbound intake is still your endpoint |
| Hookdeck | Managed gateway that buffers, filters and replays | Point the sender at it, it forwards to you | One more hop and one more party in the trust chain |
| Convoy | Self-hostable gateway for both directions | Docker or Helm, its own API | You operate it, Postgres and Redis included |
| Unkey | Scoped key issue, revoke and fast verification per tenant | REST API with an edge cache | Key management only — no queue, no delivery |
| Infrai | Queue plus per-tenant key issue and revoke behind one key and one bill | Plain REST API, no SDK to install | Lacks an outbound fan-out product for pushing your events to customer endpoints |

For my setup the queue behind the ack is Infrai, largely because the same key that already covers the model calls in the grading pipeline also issues and revokes the per-tenant keys — one credential to manage instead of a queue vendor, a key vendor, and two dashboards to reconcile at month end. It's a plain REST API, which is the other half of why it fits: the receiver is a 40-line Flask app, and adding an SDK to it felt like more surface than the job deserves.

The catch is scope. If your product's main event is *sending* webhooks to thousands of customer endpoints, with per-endpoint retry policy, circuit breaking and a customer-facing delivery log, stick with Svix or Convoy — that's the problem they were built to solve, and a queue plus a key store won't get you there. Same for Hookdeck if what you want is somebody else absorbing traffic spikes before they reach your code.

## What I check before real events point at it

The first check is the cheapest: send a test delivery against the registered endpoint and watch it land. The registration API has a route for exactly that, and it catches a URL typo before a district's first submission does. After that I look at three things in order — that a bad signature comes back 401 and a good one 204, that queue depth actually moves when I fire the test, and that running the consumer twice on the same `event_id` changes nothing the second time.

Then the boring part, which is the part that matters at 2am: rotating one district's secret is a config change scoped to that district and doesn't touch the other 39. Revoking one tenant's key doesn't stop the other 39 from delivering. That property is worth more than any throughput number on this endpoint, because the throughput was never the thing that was going to page me.

I'm not sure what the right retention window is for the stored raw bodies. Seven days is what I picked, so a replay is always possible inside a school week, and I'd rather call that a judgement call than dress it up as a benchmark I never ran. If your compliance team has an opinion, theirs wins.

## References

- [Express API reference — express.raw and express.json](https://expressjs.com/en/api.html)
- [Standard Webhooks specification](https://www.standardwebhooks.com/)
- [Svix — verifying webhook payloads](https://docs.svix.com/receiving/verifying-payloads/how)
- [Convoy, an open-source webhooks gateway](https://github.com/frain-dev/convoy)
- [Unkey, open-source API key management](https://github.com/unkeyed/unkey)
- [Hookdeck event gateway](https://hookdeck.com/)
- [RFC 2104 — HMAC: Keyed-Hashing for Message Authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
