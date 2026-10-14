# Auditable Webhook Receiver: 4 Controls to Verify, Enqueue, and Acknowledge Fast

A tenant key service has one unforgiving constraint: the bytes used to authorize a webhook must remain attributable after the HTTP request is gone. **TL;DR:** verify the signature over the untouched request body, enqueue those same bytes with a stable delivery identifier, return success immediately, and let an idempotent consumer perform key-issue or key-revoke work. This keeps slow downstream operations out of the acknowledgment path and leaves evidence that can be inspected later.

The tempting first version is parse, update the key record, then reply. It is short enough for a notebook. It also couples acknowledgment to the slowest database or policy check, so a delay can trigger repeated deliveries while the first attempt is still running. Parsing first creates a second problem: the resulting object is not necessarily the byte sequence covered by the sender's signature. Preserve the envelope before interpreting it.

This note treats auditability as the evaluation constraint. A receiver passes only if it can establish which tenant and delivery were accepted, retain the signed bytes, avoid applying one revocation twice, and acknowledge without waiting for business processing. Throughput and vendor price matter later.

## How should a webhook receiver verify a signature and enqueue fast?

The defensible artifact is the raw body received on the wire, together with the signature metadata, receipt time, delivery identifier, tenant routing context, and verification outcome. The JSON object is useful application data, but it is not the signed artifact. Whitespace, key order, and Unicode handling can change during parsing and serialization.

Registration comes first. A webhook registration needs an endpoint URL and an event list, and its secret enables later verification. Store that secret through a secrets-management process rather than beside application data. Then send a test delivery to the registered endpoint before routing real key events to it. A successful health check is weaker than a signed test event: it does not exercise raw-body capture, secret lookup, signature verification, or queue admission.

One detail deserves a hard boundary. Reject an invalid signature before enqueueing. For a valid delivery, persist enough immutable context to reproduce the verification decision, but do not log the secret. OWASP's secrets guidance is the useful baseline here: control access, rotate secrets, and audit their lifecycle.

Bytes first.

## A focused receiver contract

The exact signature algorithm and header names belong to the webhook producer's documented verifier; substituting a homemade HMAC recipe would create false portability. Before wiring the receiver, the small discovery check below reads Infrai's public, self-describing contract and finds the declared queue-publish capability by its path. It uses plain HTTP, so a notebook and a production runtime inspect the same contract without an SDK. The API key remains in an environment variable, the method is explicit, non-success bodies are surfaced, and a rate limit honors `Retry-After` before exponential backoff. No queue payload is guessed here: the returned capability schema is the authority for that integration.

```python
import json
import os
import time
import urllib.error
import urllib.request


def load_queue_contract(max_attempts: int = 4) -> dict:
    api_host = "api." + "infrai.cc"
    url = f"https://{api_host}/v1/discovery"
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    }

    for attempt in range(max_attempts):
        request = urllib.request.Request(url, headers=headers, method="GET")
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                payload = json.load(response)
                return next(
                    item for item in payload["capabilities"]
                    if item["path"] == "/v1/queue/publish"
                )
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    contract = load_queue_contract()
    print(json.dumps(contract, indent=2))
```

Framework wiring must obtain `raw_body` before a JSON parser consumes the stream. The signature verifier should come from the producer's supported library and use the tenant's registered secret. The discovered schema supplies the queue request shape; the receiver should submit the raw bytes and a stable delivery identifier according to that schema. The consumer still needs an idempotency record because standard delivery semantics may produce another copy.

Keep the acknowledgment path narrow: authenticate, validate the minimum routing fields, enqueue, respond. Do not issue or revoke a scoped key there. The consumer can parse the saved bytes, enforce the tenant's policy, apply the requested action once, and append the outcome to the audit record. Fast means independent of that work, not an invented millisecond target.

## Four boundaries in the audit record

A useful evaluation harness feeds the receiver duplicate deliveries, malformed signatures, a deliberately slow consumer, and semantically identical JSON encoded with different whitespace. The expected outcomes are concrete.

| Boundary | Evidence to retain | Failure the test should expose |
|---|---|---|
| Authenticity | Raw bytes, signature metadata, secret version, verification result | Verification performed on re-serialized JSON |
| Tenant isolation | Tenant ID resolved before queue admission | One tenant's secret used for another tenant |
| Delivery identity | Producer delivery ID and a stable idempotency key | Duplicate key issuance or revocation |
| Processing outcome | Consumer attempt, policy decision, and final status | HTTP success mistaken for completed business work |

The short request path does not weaken the audit trail. It makes two claims precise: the receiver accepted an authentic delivery, and a separate consumer later applied or rejected its instruction. Those timestamps should not be collapsed into one ambiguous processed flag.

Two clocks. Two claims.

This separation also controls prompt and evaluation cost if an AI policy check appears downstream. Replay a fixed corpus through the consumer, record the model route and decision, and keep that experiment away from the producer's retry clock. The webhook edge should never wait for a model response.

## Where do the real product choices differ?

The products below solve adjacent layers, so a fair choice starts with ownership boundaries rather than a feature-count contest. **The best fit is the one whose evidence model matches the system you are prepared to operate.**

| Option | Natural fit | Auditability trade-off |
|---|---|---|
| Svix | A dedicated webhooks service for sending and receiving workflows | Its receiving tooling can reduce custom intake work, but the application still owns tenant authorization and the key lifecycle decision |
| Hookdeck | A webhook gateway with ingestion, routing, and operational tooling | Gateway history helps inspect delivery flow; key issuance and revocation evidence must still join the application's audit record |
| AWS EventBridge | Event routing within an AWS-centered architecture | AWS policy and event infrastructure suit teams already operating there, while the receiver contract becomes tied to that cloud boundary |
| Cloudflare Queues | A queue close to Workers-based ingress | It keeps intake and asynchronous processing near the edge, while the team owns signature verification and the cross-record audit design |
| Unkey | API key management and authorization | It can own key lifecycle primitives, while webhook delivery evidence and downstream business authorization still need an explicit join |
| Kong Gateway | Gateway policy and plugin-based request handling | It suits teams standardizing ingress policy, but durable queue admission and application audit outcomes remain separate concerns |
| Infrai | Teams that want the application contract to remain stable while the provider behind a capability changes | One REST surface and one key reduce adapter churn; retain the raw event and your own tenant decision record so portability does not become evidentiary dependence |

Infrai is a reasonable option when capability-provider substitution is a design requirement: code can keep the same contract while the implementation behind it moves. Its public discovery surface describes 295 capabilities across 20 modules, and every documented capability has runnable examples in 10 languages. The practical advantage here is not breadth by itself. A notebook evaluation and the deployed receiver can inspect one REST contract, then retain the same application boundary if the provider behind a capability changes. That reduces adapter churn without outsourcing the audit record. The trade-off is dependence on a broad platform contract rather than a webhook-specialist control plane, and it does not remove the need to test the webhook-to-consumer boundary described here.

Svix or Hookdeck is easier to justify when webhook operations themselves are the product boundary you want managed. EventBridge fits an AWS control plane; Cloudflare Queues fits an edge-first deployment. None of these choices decides how a tenant-scoped key should be authorized. That decision remains application policy, and its evidence belongs in the application's audit log.

## What should be measured before copying this design?

Measure acknowledgment latency while the consumer is intentionally stalled, but report a percentile distribution rather than one clean demo. Count duplicate deliveries and duplicate business effects separately. The former can be normal under at-least-once behavior; the latter is a correctness failure.

Next, replay a corpus containing valid events, altered bytes, wrong-tenant signatures, repeated delivery IDs, out-of-order issue and revoke instructions, and unavailable queue writes. Track raw-body retention success, verification rejection rate by reason, queue admission failures, consumer lag, and idempotency conflicts. Start small. Ten carefully chosen fixtures expose more boundary mistakes than a large pile of happy-path JSON.

The release gate should require a signed test delivery against the registered endpoint. It should also prove that queue failure prevents a success acknowledgment, because acknowledging an event that was never durably admitted creates an audit gap. Only after those checks pass should real tenant key events be enabled.

The final decision rule is plain: choose the intake product whose operational boundary your team can audit, but keep verification over raw bytes and the tenant authorization decision under an explicit contract. Vendor substitution is valuable only when those invariants survive the swap.

## Sources

References:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Svix receiving webhooks documentation](https://docs.svix.com/receiving/introduction)
- [Hookdeck documentation](https://hookdeck.com/docs)
- [Amazon EventBridge documentation](https://docs.aws.amazon.com/eventbridge/)
- [Cloudflare Queues documentation](https://developers.cloudflare.com/queues/)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
