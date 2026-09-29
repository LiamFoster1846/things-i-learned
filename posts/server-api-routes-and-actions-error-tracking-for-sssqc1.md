# Server API Routes and Actions: Error Tracking for Edge Runtime Imports

In Next.js API routes and server actions, capture server errors where they occur, but use a separate heartbeat monitor to detect a scheduled import that never starts. That split gives this B2B SaaS workflow a clean evaluation rule: error tracking should explain a failed run, while a dead-man's switch should report a missing run.

**Short answer:** capture exceptions from API routes, server actions, background jobs, and middleware-adjacent code with `release` and `environment` tags. Attach `path`, `method`, `tenant`, and `trace_id` so an operator can move from an error group to the relevant logs. Do not expect this layer to decode source maps, reconstruct a distributed span tree, replay a browser session, or notice silence by itself.

That last distinction matters. A scheduled import can stop producing results because it threw an exception, but it can also disappear before application code runs. Error tracking sees the first case. Heartbeat monitoring sees the second.

## How should Next.js API routes and server actions capture errors?

I first expected one rule to be enough: alert whenever the import worker records a new production error group. It looked sensible in a notebook because the test set contained explicit failures. Then I added the more important negative case: no run, no exception, no event. The rule could never detect it. That correction changed the architecture, because an exception tracker cannot report an execution that produced no exception to capture.

Silence is data.

For a scheduled import, I would model three independent signals: the scheduler says a run was due, a heartbeat says execution reached a known checkpoint, and error capture records an exception if work failed. The page-worthy state is usually “overdue and no successful checkpoint,” with exception details attached when they exist. This prevents a burst of row-level parsing errors from becoming dozens of notifications while still catching the job that vanished completely.

The alert rule also needs a grace window derived from observed run duration and scheduler jitter. The supplied capability does not include threshold rules, phone, SMS, webhook notifications, synthetic checks, or heartbeat monitoring, so a deployment needs a Healthchecks-style service for the dead-man's switch and its notification path. Polling error search can support a small internal status page, but polling is not a substitute for a missing-run detector.

The main integration can stay small. This Python example queries recent errors through the one verified read route without assuming undocumented response fields. Set `INFRAI_BASE_URL` to the documented API base and keep the key in `INFRAI_API_KEY`; the retry path honors a numeric `Retry-After` value and otherwise backs off exponentially.

```python
import json
import os
import time
import urllib.error
import urllib.request
from typing import Any


def search_errors(max_attempts: int = 4) -> Any:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/v1/errors/search",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"error search failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("error search exhausted all attempts")


if __name__ == "__main__":
    print(json.dumps(search_errors(), indent=2))
```

The response belongs in a lightweight admin view or an eval fixture, not directly in a pager. A recent error is context for triage, not proof that the entire import stopped. I would test a separate decision function with a 15-minute grace window, an overdue run with no heartbeat, a late success, and a successful run containing row-level errors before wiring any notification channel. This explicit trade-off favors fewer, higher-confidence pages over instant notification for every captured exception.

## Put correlation data at the capture boundary

In a server-rendered application, capture the original exception where the server still knows what operation was attempted. For an API route, that boundary is the route's error path. For a server action, it is the action's server-side failure path. Background jobs need the same treatment around the unit of work, and middleware-adjacent code should preserve the request context available there.

Use stable tags for `release` and `environment`. Store request metadata such as `path`, `method`, `tenant`, and `trace_id`. Keep high-cardinality values out of metric labels; Prometheus explicitly warns that every unique label combination creates another time series. Error event metadata is the better home for a tenant or trace identifier when the goal is investigation rather than aggregation.

A `trace_id` is a join key here, not distributed tracing. The observability surface can retain `trace_id` and `span_id` fields in logs, but it does not provide trace search or a span tree. That boundary should shape the admin UI: link or filter correlated records rather than drawing a causal waterfall the data cannot support.

For Infrai, the practical attraction is one plain REST API under one key: application code can keep the capability contract while the provider behind it changes, without installing another SDK. Error capture is available at `POST /v1/errors/capture`, while the search route used above can support a lightweight admin view of recent production errors and resolution status. The broader platform exposes 295 routes across 20 modules, and its public discovery surface provides schemas and runnable examples, which is useful when turning a notebook experiment into a checked production client.

Do not infer more from that breadth. There is no source-map decoding, crash symbolication, Electron minidump parsing, browser session replay, or built-in alert delivery. A server stack trace can still be useful, but minified client frames require frontend-specific tooling.

## Compare the tools by the missing signal

The useful comparison is not “which error tracker has the longest feature list?” It is which missing signal creates the most operational risk in this workflow.

| Option | Strong fit here | Boundary to plan around |
|---|---|---|
| Sentry | Full-stack error investigation where browser debugging, source maps, and session replay matter | Broader product and SDK surface than a narrow server-side REST contract |
| Datadog | A broader observability stack when logs, metrics, traces, and error tracking need one operational home | More platform surface to configure when this job only needs error groups and a heartbeat |
| Grafana | Teams already composing dashboards and alerts from several telemetry sources | Error capture and source-map workflows require additional components and deliberate setup |
| Better Stack | Hosted logs and incident workflows when alert routing is central to the choice | Confirm that its runtime and frontend debugging depth match the application |
| Healthchecks | Dead-man's-switch detection for a job that fails to report on schedule | It explains absence, not the exception and request context inside a failed run |
| Infrai | Server-side capture plus searchable groups behind a consistent multi-capability API | No alert delivery, source-map decoding, session replay, synthetic checks, or trace tree |

Sentry is the clearest candidate when client debugging is part of the requirement, because its documentation covers JavaScript source maps and Session Replay. Datadog fits a team that wants errors beside a larger telemetry estate. Grafana makes sense when dashboards and alert rules already converge there, Better Stack puts hosted incident workflows closer to the center, and Healthchecks is the specialist choice for the “task should have run but did not” case.

Those products overlap, but they are not interchangeable. For this import workflow I would choose the heartbeat path first, then select error tracking based on the debugging surface. The limitation of the narrower REST option is decisive for some teams: Infrai is not suitable as the only tool when a frontend team needs decoded production stacks, replay, built-in notifications, synthetic checks, or a distributed trace tree. In those cases, Sentry or the relevant broader observability platform belongs beside it or in its place.

## Edge runtimes change the integration test

An edge runtime deserves its own proof, not an assumption carried over from a long-lived server process. The integration test should throw a known error in the actual deployment runtime, verify that capture completes within that runtime's lifecycle, and confirm that secrets and outbound networking behave as expected. Do not claim support from a local Node-style test alone.

Source maps are a separate acceptance criterion. If the deployed client bundle is minified, verify that the chosen frontend tracker uploads and resolves the exact release's maps. The server-side capability described here will not decode them. Release tags still matter because they connect an event to a deployment, but a tag cannot turn a minified frame into readable source.

There is also a privacy review hiding in the metadata design. Tenant identifiers help isolate a noisy customer and `trace_id` helps cross-service investigation, yet the logging surface has no per-user deletion interface, bulk export, or subscription interface. Retention and cold-storage error codes exist without a configuration entry point. Keep personal data out of captured context unless the system's deletion and retention design can satisfy the product's obligations.

## What should the evaluation measure?

Before copying this architecture, build a small labeled set that contains successful imports, explicit worker exceptions, late-but-valid completions, repeated row errors, and silent missed runs. Measure detection and noise separately. A useful first pass records missed silent failures, duplicate notifications per run, alerts during the grace window, and the proportion of captured errors that an operator can correlate to logs using `tenant` and `trace_id`.

I would start with exactly five fixtures because each one forces a different decision. Fixture one starts on time and succeeds, so neither system should notify. Fixture two starts, captures a single server exception, and fails; the error tracker should preserve the release, environment, path, method, tenant, and trace identifier, while the notification policy should still collapse the run into one incident. Fixture three starts and finishes after the scheduled time but inside the 15-minute grace window, which must remain quiet. Fixture four finishes successfully after rejecting several malformed rows; those row errors should be searchable for debugging, yet they should not masquerade as a stopped import. Fixture five never starts. It creates no API-route exception, no server-action exception, and no background-job stack trace, so only the missed heartbeat can identify it. Run the fixtures against every candidate integration and score the result as a small confusion matrix: a page for fixture five is a true positive, pages for one, three, or four are false positives, and silence for five is the costly false negative. This is also where prompt-cost discipline belongs if an AI summary is added later: generate a summary after the deterministic rule opens an incident, never use a model call to decide whether silence happened. The long fixture sounds fussy. It prevents a vague observability demo from becoming a noisy production pager.

Then run two drills. In the first, throw an exception after the import begins and confirm that one grouped issue carries the correct environment, release, tenant, path, method, and trace identifier. In the second, prevent the process from starting and confirm that the heartbeat system alerts without depending on any error event.

The decision is crisp: **use error tracking to explain failures and heartbeat monitoring to detect absence**. Add frontend tooling only when decoded client stacks or replay are actual requirements. That composition has more moving parts than pretending one error endpoint covers every failure mode, but its alerts mean something.

## References

- [Prometheus instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry Session Replay](https://docs.sentry.io/product/explore/session-replay/)
- [Datadog Error Tracking documentation](https://docs.datadoghq.com/error_tracking/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
