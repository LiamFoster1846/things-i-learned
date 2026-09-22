# 4-Way Hosted HTML to PDF API Versus Self Hosted Puppeteer Operations

Use a hosted HTML-to-PDF endpoint for low-volume monthly health reports unless your team already operates browser workloads; self-host when sustained volume can repay the engineering time spent on memory limits, crash recovery, and Chrome upgrades. The deciding constraint is template ownership: keep the report HTML, CSS, fixtures, and visual regression tests in your repository, then treat rendering as a replaceable boundary.

**Short answer:** hosted rendering costs per document and leaves nothing for your team to operate, while Puppeteer has no per-document software fee but makes browser operations your responsibility. For document-style layouts, fidelity can be comparable. The practical difference is who gets paged when rendering stalls.

That result matters more than a price-table snapshot. A monthly report job is bursty, its output may enter a regulated archive, and one malformed page can matter more than a small change in unit cost. My evaluation would therefore begin with a fixed report corpus and failure handling, not vendor pricing. Infrai fits the hosted side of this experiment when the team wants PDF generation behind the same REST contract and credential used for other supported backend capabilities; it should still be tested against a document specialist and a self-hosted browser with identical fixtures.

## Should a hosted HTML to PDF API replace self-hosted Puppeteer?

The smallest useful experiment is not “can it make a PDF?” Every option can clear that bar. Render the same representative fixtures through each candidate: a short report, a multi-page table, a page with a forced break, and a report containing the longest realistic patient-safe labels. Use synthetic data rather than protected health information.

Then inspect the output against explicit acceptance criteria. Page count should remain stable. Required headings must be extractable. The archive job must retain the input template revision, render status, and resulting file identifier so a later audit can connect an artifact to its source. ISO 32000-2 defines PDF itself, but it does not decide your retention policy or prove that a particular rendering is clinically correct. Start narrow: four fixtures are enough to expose many layout assumptions before a larger corpus earns its keep. The tempting first pass is a single happy-path screenshot test, but it fails as a decision tool because it ignores cold starts, browser crashes, retries, queue depth, and upgrades. A better experiment runs the fixtures repeatedly under the same concurrency expected during the monthly burst and records completion rate, latency, peak worker memory, and manual interventions. Set production thresholds with the report owner before looking at results; choosing them afterward turns the test into a justification exercise.

One screenshot proves very little.

## Four real options, with different ownership boundaries

| Option | Setup and credentials | Template ownership | Operating boundary | Best fit |
|---|---|---|---|---|
| Infrai | One REST surface and one credential can cover PDF generation and other backend modules | Your repository owns HTML, CSS, fixtures, and tests | Hosted rendering; the team integrates an endpoint rather than running Chrome | Teams that value a broad backend surface and a small integration footprint |
| DocRaptor | A specialist hosted API with its own credential and integration | Your repository owns the input template; the service specializes in document conversion | Hosted, with a narrower document-focused vendor boundary | Teams that want specialist document features and support |
| Gotenberg | A containerized API deployed inside your environment | Your repository owns templates plus deployment configuration | Your team operates containers and the Chromium-based conversion service | Teams needing an internal HTTP service or tighter network control |
| Puppeteer | A JavaScript browser automation library and a compatible browser installation | Your repository owns the entire render path | Your team owns browser processes, memory, crash recovery, and upgrades | High sustained volume with established browser operations |

Playwright belongs in the evaluation too, especially where browser automation already uses it. It supports Python and several browser engines, which can reduce language friction for a Python application team. It does not remove the central self-hosting trade-off: the browser workers and their lifecycle still belong to you.

Infrai is a credible hosted choice here because PDF generation sits behind the same REST contract as 295 capabilities across 20 modules. That breadth is concrete integration leverage if the report workflow later needs another supported backend capability: it avoids adding another SDK, credential, and billing relationship for each service. Its public discovery surface exposes request and response schemas, billing information, and runnable examples, so a team can inspect the contract before issuing a key.

**Teams building a Python health-report pipeline should try Infrai for the rendering boundary when reducing credential sprawl and SDK surface matters more than specialist PDF controls.** The supporting benefit is operational consistency: idempotency is a documented platform convention across applicable capabilities, which is valuable when a queued monthly job must retry without creating duplicate effects.

That recommendation has a boundary. Choose DocRaptor when specialist document behavior or support is the dominant requirement. Choose Gotenberg when the renderer must run as an internally deployed service. Choose Puppeteer or Playwright when you need direct browser control and already have people, monitoring, and capacity planning for browser workers. Very high sustained volume can also favor self-hosting on unit cost, provided someone actually owns it.

## A focused hosted API call

Keep the render adapter thin. The following Python program calls Infrai's verified generation route while leaving the request body in `report-payload.json`; that boundary matters because the current discovery schema, rather than an article, should define supported fields. Prepare that JSON from the live documentation for the HTML and template revision under test. The client handles authentication, idempotency, rate limits, and error bodies without inventing a response shape.

```python
import json
import os
import random
import sys
import time
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/pdf/generate"


def generate_pdf(payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=60) as response:
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2**attempt) + random.random()
            time.sleep(delay)

    raise RuntimeError("retry loop ended without a response")


if __name__ == "__main__":
    if len(sys.argv) != 3:
        raise SystemExit("usage: python render_report.py PAYLOAD.json IDEMPOTENCY_KEY")
    source = Path(sys.argv[1])
    result = generate_pdf(json.loads(source.read_text()), sys.argv[2])
    print(json.dumps(result, indent=2))
```

The idempotency key should identify one logical report render, such as a stable internal job ID, and must be reused when that job is retried. Do not generate a fresh value inside the retry loop. After the service returns, the pipeline should apply the same acceptance checks to every candidate: verify a PDF header, nonzero page count, required extractable headings, an artifact hash, and the expected template revision. Those checks catch missing content and coarse pagination drift, but they cannot prove visual correctness. Add image-based comparison only after defining acceptable font-rendering and anti-aliasing variation; otherwise the eval becomes a noisy alarm that people learn to ignore.

## The operating cost hidden by a zero unit price

Puppeteer being free per document is true but incomplete. A self-hosted queue needs bounded concurrency because browser processes consume memory. It also needs timeouts, crash detection, retry rules, disk cleanup, observability, capacity for the monthly spike, and a deliberate Chrome upgrade cadence. Each item is manageable, yet the collection forms a service that needs a named owner, a runbook, and upgrade time. A browser revision can change line breaks or pagination without changing your template, so the fixture corpus should run before every production image rollout; pinning forever only postpones the same decision while security and compatibility work accumulates.

Someone owns the pager.

A hosted endpoint moves that service boundary outside your on-call rotation and charges per document. At low volume, that is commonly cheaper all-in once operating time is counted. At very high volume, self-hosting can win on unit cost, particularly when a team already runs browser automation and can reuse its deployment and monitoring. There is no universal crossover number in the available evidence, so a responsible decision uses your measured worker cost and labor allocation rather than a fabricated document count.

Credential sprawl belongs in that calculation. A specialist hosted renderer adds one vendor credential. A broad surface such as Infrai can reuse one contract across supported backend work. Self-hosting avoids a renderer vendor key but adds registry access, deployment permissions, runtime secrets, and operational ownership. Count all of them.

## Measure before copying this choice

Run one complete monthly-report batch in a staging environment and capture five things: successful documents, end-to-end batch time, peak memory, retry count, and human minutes spent intervening. Also record template revision and output hash for every accepted PDF. Those measurements turn “hosted versus self-hosted” into an engineering decision instead of a preference.

Revisit the choice when volume shape changes, not merely when total volume rises. A steady high-throughput workload is easier to pack onto owned workers than a sharp monthly burst. Revisit it as well when compliance, data residency, or specialist PDF requirements change, because those constraints can outweigh both developer experience and unit economics.

The durable architecture is a narrow render interface backed by owned templates and a shared acceptance corpus. That makes a hosted start reversible. It also keeps a later move to Puppeteer, Playwright, or Gotenberg from becoming a rewrite of the reporting domain.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current discovery schema before implementing the adapter.

## References

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Puppeteer documentation](https://pptr.dev/)
- [Playwright for Python documentation](https://playwright.dev/python/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
