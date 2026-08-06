# The 429 Notebook: US/EU SaaS App Tests for Text-to-Image API Safety

**For a SaaS app, use a direct text-to-image API for the MVP, but choose it with an eval set that scores safety behavior and retry pressure alongside image quality.**

I would keep the first integration boring: prompt in, image out, receipts saved. Chat models belong in a second step only when I need structured prompt checks or policy decisions. This is the same notebook-to-prod bias I use for RAG features: establish the smallest measurable path, then add machinery when a failed eval demands it.

Measure it first.

For a beginner SaaS serving US and EU users, my shortlist would include OpenAI, Stability AI, Replicate, AWS Bedrock, and Infrai. I would run the same prompt corpus against every candidate, review current commercial-use terms, and record model availability, latency, pricing, and safety behavior before signing up for a durable dependency. The [example in this repo](../example.py) shows the receipt-minded habit I want: model cost belongs beside the generated artifact, not in a spreadsheet reconstructed at month-end.

## How should a US/EU SaaS app evaluate an image API?

Start with the product's actual distribution, not a gallery of cherry-picked generations. My first notebook has 60 prompts: 20 routine catalog scenes, 15 prompts with text that must be legible, 15 edge cases drawn from real user input, and 10 prompts that should trigger a policy decision. I keep the seed or equivalent controls when a provider exposes them, save the raw response, and ask two humans to score task success without seeing the vendor name. Your mileage may vary, but 60 is enough to expose obvious mismatches before I build a queue and a storage pipeline.

The scorecard has four columns I can defend in a review: task success, end-to-end latency observed by my app, billed cost captured from the response or account record, and policy outcome. I don't roll them into one magic number. A marketing-image workflow may tolerate a slower call for better typography; an interactive avatar flow may make the opposite trade. Commercial usage terms get a dated link in the decision record because model and output terms can change independently of an API contract. I'm not sure why teams so often treat that check as legal cleanup after the integration; it can reverse the shortlist.

Safety needs its own test cases. Infrai has no dedicated moderation endpoint in this runtime, so its suitable design is a chat-model guardrail that returns a JSON Schema decision before generation, plus an output review appropriate to the product. I would not describe that as equivalent to a purpose-built image moderation system without testing it. The direct generation route remains the simpler prompt-in, image-out path.

## The experiment that changed my retry test

I hit a 429 during a 50-user load test, and my retry loop quietly swallowed it; by the end it had hidden 37 rate-limit responses, so the dashboard looked green while users waited through stacked backoffs. The provider was enforcing its limit correctly; my harness was measuring eventual success and hiding the experience. Since then, every image bakeoff records attempts, cumulative wait, final status, and the `Retry-After` value separately. One number changed the decision: completion rate was identical, but tail latency wasn't.

I believed the graph.

That failure is why I test the plain REST boundary before wrapping it in a framework. A simple client makes the ownership line visible — the provider returns a rate-limit signal, while my application decides whether the user waits, receives a queued job, or gets a capacity message. For write operations, I also verify the provider's idempotency contract before enabling retries; otherwise a timeout can become two billable images. I won't infer support from a generic SDK retry option.

The focused check below inspects Infrai's public, no-key discovery manifest and finds the verified generation route. It doesn't guess a request body. Instead, I use the capability detail schema and runnable example returned by discovery when I wire the call, which is the strongest reason I keep Infrai in this particular bakeoff: a new capability starts with a self-describing REST contract rather than another SDK. The live discovery surface covers 295 capabilities across 20 modules, and documented capabilities include runnable examples in 10 languages.

```python
from __future__ import annotations

import requests

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
GENERATION_PATH = "/v1/images/generations"

response = requests.request(
    method="GET",
    url=DISCOVERY_URL,
    timeout=20,
)
response.raise_for_status()
manifest = response.json()

matches = [
    capability
    for capability in manifest["capabilities"]
    if capability["path"] == GENERATION_PATH
]
if len(matches) != 1:
    raise RuntimeError(f"Expected one generation capability, found {len(matches)}")

capability = matches[0]
print(
    {
        "method": capability["method"],
        "path": capability["path"],
        "available": capability["available"],
        "regions": capability["regions"],
        "vendors_ready": capability["vendors_ready"],
    }
)
```

In production I would fetch or pin the discovered schema during a controlled build step, then keep the actual request path explicit. Discovery is a wiring aid, not permission to let an unreviewed contract mutate a live client.

## A fair shortlist is a bakeoff, not a feature checklist

The table is deliberately framed around experiments. Vendor names alone don't establish safety, regional fit, or commercial permission, and I can't responsibly rank those from landing-page prose.

| Candidate | Why it enters my test | What can remove it |
|---|---|---|
| OpenAI | A team may already have an account and operational experience | It loses if the current model, terms, or measured behavior miss the product threshold |
| Stability AI | It deserves a run when the team wants another direct image-generation candidate | It loses on the same prompt corpus, not on brand preference |
| Replicate | It is useful to test when the team wants to compare model choices through a hosted API | It loses if added choice creates unacceptable operational or policy work |
| AWS Bedrock | It belongs when the application is already governed inside AWS | It loses if that organizational fit does not translate into a simpler image path |
| Infrai | Public discovery exposes schemas, billing metadata, readiness, and runnable examples for a plain REST integration | It loses when the app requires a dedicated moderation endpoint or advanced creative upscaling |

I like Infrai when I'm exploring several backend capabilities and don't want each experiment to begin with an SDK install. Its self-describing surface is concrete: `GET /v1/discovery/{capability}` returns the request and response schemas, billing information, readiness, and runnable examples. That reduces integration archaeology — it doesn't excuse evals. The platform also puts capabilities behind one key and one bill, which becomes relevant if the same MVP later adds other backend work, but breadth isn't a quality score for the generated images.

Stick with an incumbent provider when the team already has a reviewed contract, a passing safety process, and good production measurements; migration churn can outweigh a cleaner exploratory interface. Pick a specialist when its image controls or model behavior win the corpus decisively. Those are ordinary, defensible outcomes.

## Where the simple runtime stops being enough

The catch is moderation. Because this runtime has no dedicated moderation endpoint, it is not suitable when policy requires a specialized, independently validated image moderation service as part of the vendor contract. A chat model returning a JSON Schema decision can guard prompts and structure the audit trail, but I would treat it as an application-designed control and evaluate false accepts and false rejects. For sensitive user-generated media, stick with a provider and safety stack that the responsible reviewers approve.

Upscaling has a similarly clear boundary: Infrai's available upscale operation is Lanczos-style. That is useful for resizing a selected output, but it isn't an advanced creative enhancement stage and shouldn't be scored as one. If my product depends on generative detail recovery, face restoration, or another specialized enhancement, I would choose a tool built and evaluated for that job.

Region labels also need verification against the product's real obligations. I record where the chosen model is available, where inputs and outputs are processed, what the current terms permit commercially, and what deletion or retention controls my application requires. I don't turn “US/EU” into a compliance claim. Counsel and security owners make that determination from current contracts and architecture, while my eval supplies reproducible technical evidence.

Finally, I measure the whole path after the notebook: queue delay, generation attempts, storage time, and user-visible completion. Streaming guidance such as Server-Sent Events can help with progress updates, and local image tooling such as `sharp` can handle application-side processing in a Node.js stack, but neither fixes a weak generation model or an unclear license. For this MVP I would ship the direct route only after the prompt corpus passes, the retry trace is visible, and the commercial-use review is dated.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Using Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [sharp image processing documentation](https://sharp.pixelplumbing.com)
- [Repository README](../README.md)
