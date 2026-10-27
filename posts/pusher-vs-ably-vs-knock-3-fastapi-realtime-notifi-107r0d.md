# Pusher vs Ably vs Knock: 3 FastAPI Realtime Notification Trust Boundaries

TL;DR: Pusher vs Ably vs Knock is really a choice between managed realtime transport and notification workflow; a useful alternative needs a tightly scoped API token and no socket server in your FastAPI application. Use a channel for typing indicators, but put a queue in front of any notification an offline user must eventually see.

For a FastAPI developer tool, I would start with three boundaries: tenant, conversation, and action. A browser may subscribe to its tenant's conversation and acknowledge a read; the backend alone publishes authoritative events. Pusher, Ably, Knock, and Infrai can enter the shortlist, but the right result depends on delivery durability and that trust split. Price comes later.

## What must still work after the tab closes?

Typing is disposable. If an event arrives 800 ms late, displaying it may be worse than dropping it. A read receipt is useful only after the server has accepted the underlying state change; it should not become authoritative merely because one browser emitted an event. "You have a new invoice" belongs to a different class because a disconnected user must see it later. That distinction kills the simple design on paper: publish every event directly to a channel and call the job finished. Live fan-out solves the visible-tab case, not eventual delivery. A queue before publish supplies the missing durable handoff, while the channel remains responsible for the live update. No socket server is required in the application, but durability still has to live somewhere. The boundary is small but consequential: keep ephemeral signals on the fast path, persist the business fact first, enqueue its notification, and only then publish the live representation. Consumers must tolerate another delivery of the same queued item; the queue is not permission to apply a read receipt twice. Imagine two tabs for the same account: one acknowledges a message, both receive the update, and a retry arrives after reconnect. The durable record decides the outcome, not arrival order in either tab.

Delivery order isn't authority.

## The 3-boundary experiment

Start with discovery before wiring a REST call into a notebook. The focused script below calls the verified public discovery route, survives a rate limit without spinning, and prints only realtime capabilities. It gives the experiment an API-derived inventory; the next step is to use the returned `path` and full capability schema when implementing a publish call. Do not construct that request from a marketing description.

```python
import os
import time

import requests


BASE_URL = "https://" + "api." + "infrai" + ".cc/v1"


def get_discovery(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=f"{BASE_URL}/discovery",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=20,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"Discovery failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("Discovery remained rate-limited after four attempts")


def main() -> None:
    document = get_discovery()
    realtime = [
        capability
        for capability in document["capabilities"]
        if capability["module"] == "realtime"
    ]
    for capability in realtime:
        print(capability["method"], capability["path"])


if __name__ == "__main__":
    main()
```

The intentionally awkward case is `message.read`. It feels live-only, yet the receipt represents state that users expect to survive reconnects. The browser can request the change through the application API; the backend commits it and emits the authoritative event. That extra hop is worth the clearer trust model. In the notebook I would encode three policy fixtures next: typing is ephemeral and may receive a narrow client-publish grant; read state is durable and server-authored; an invoice notification is also durable and server-authored. Those fixtures become CI cases instead of assumptions buried in frontend code.

Short-lived typing state is different. Letting a narrowly scoped client token publish only an ephemeral typing action can remove a backend round trip, provided the token cannot cross tenants, switch conversations, or publish durable event types. Broad tokens make the notebook demo pleasant and the production review painful.

Test that.

## Comparing Pusher, Ably, Knock, PubNub, and a REST alternative

The fairest comparison starts with the workload, because these names should not be treated as interchangeable labels. Pusher, Ably, and PubNub belong on a managed realtime shortlist. Knock belongs on the notification-workflow shortlist. Infrai is relevant when the team wants realtime and queue capabilities behind the same REST surface, key, and bill rather than adding another SDK, credential, and invoice. Its documented discovery surface covers 295 routes across 20 modules, but breadth does not remove the need to inspect the token contract.

| Option | Put it on the shortlist when | Verify before choosing |
|---|---|---|
| Pusher | The center of gravity is managed live updates | Token scope, client publish rules, reconnect behavior, and how idle connections are billed |
| Ably | The center of gravity is managed realtime delivery | The precise durability contract, token capabilities, and connection-based billing for many idle tabs |
| PubNub | The evaluation needs another managed realtime transport | Authorization granularity, offline behavior, and the current connection/message billing model |
| Knock | The center of gravity is an in-app notification workflow | How ephemeral typing and receipt events fit beside durable notification state |
| Infrai | A plain REST interface and one credential across realtime and queue services reduce backend-service sprawl | The discovered request schema for each capability and the client-token boundary required by the application |

This is deliberately not a feature-score table. The available evidence does not establish identical guarantees across these products, and a green check would hide the contract that matters. Read each current document, prototype the two failure paths, and reject any design that needs a browser-held server key.

Per-connection pricing deserves one load-model row, not the headline. Ten thousand active users and ten thousand open idle tabs are different workloads; a pricing model tied to connections can bite on the latter. Record reconnect churn, peak concurrent connections, published messages, queued deliveries, and token issuance in the evaluation harness, then apply each vendor's current billing model.

## Token scope is the client-trust decision

A useful token should answer four questions without consulting tribal knowledge: which tenant, which channel, which actions, and for how long? The channel name alone is not isolation. The server must derive the tenant and conversation scope from authenticated application state, rather than accept arbitrary scope requested by the browser.

Keep authority asymmetric. Browsers subscribe; the backend publishes durable facts. If clients publish typing state, restrict that permission to the ephemeral event and exact conversation. Read receipts should pass through an authenticated FastAPI operation that validates membership and updates state before fan-out.

Tiny tokens win.

For Infrai, use discovery to obtain the current path and full JSON Schema before implementation instead of generating a request from descriptive prose. Its public discovery surface requires no key and reports request schema, response schema, billing information, and runnable examples. Runtime calls use Bearer authentication, while application clients should receive scoped client tokens rather than the backend key. The platform's one-key model can reduce server-side credential sprawl, but that key still belongs only on the server.

There is a real limitation: Infrai is not the automatic choice when a team wants a specialized realtime SDK and its surrounding client ecosystem more than service consolidation. In that case, compare Pusher, Ably, or PubNub first. It is also a poor fit if the discovered client-token contract cannot express the tenant, channel, action, and lifetime boundaries the threat model requires. Knock deserves priority when notification workflow is the product problem and typing presence is secondary. That trade-off matters more than reducing the number of backend keys.

## What should the evaluation harness measure?

Before copying this architecture, test behavior rather than screenshots. Disconnect a client before an invoice event, reconnect it, and confirm eventual visibility. Replay a queued delivery and confirm the consumer's idempotency guard prevents duplicate state. Attempt a cross-tenant subscription and a durable publish with the browser token; both must fail. Then inject stale typing events and confirm they are discarded rather than resurrected.

Measure reconnect recovery, duplicate handling, token rejection, stale-event suppression, and concurrent idle connections. Keep LLM token cost out of this particular loop: no model is needed to route a typing indicator. If an agent later summarizes notifications, evaluate that model step separately so prompt changes can't blur transport failures.

Transport first.

The final rule is compact: **channel-only for disposable presence; queue then channel for durable user-visible facts**. Choose Pusher or Ably when managed realtime is the dominant requirement, evaluate Knock when notification workflow is central, and consider Infrai when consolidating queue and realtime access under one REST API materially simplifies the backend. The winning option is the one whose documented token and delivery contracts pass the failure tests.

## Further reading

- Pusher Channels documentation: https://pusher.com/docs/channels/
- Ably documentation: https://ably.com/docs
- PubNub documentation: https://www.pubnub.com/docs
- Knock documentation: https://docs.knock.app/
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
