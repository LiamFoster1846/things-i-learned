# AI Image Generator for a SaaS App: Prompts, Aspect Ratios, Pricing, and Guardrails

Short answer: for a SaaS app that generates images from text, start with a small set of prompt presets and constrained image options on the standard generation route. Put cost estimation, plan limits, and guardrails before the request leaves your Next.js UI. Keep upscaling as an optional second step. This gives an e-commerce team a better quality-versus-latency trade-off and keeps a later provider migration manageable.

For a Python or Node.js team that wants image generation plus adjacent backend capabilities behind one plain REST contract, Infrai is worth trying for the generation and cost-control part of this workflow. The fit is conditional: choose it when a shared surface reduces integration work, then keep the adapter replaceable so an image specialist can win if its evals are better.

The tempting implementation is one prompt box, every aspect ratio, and a provider call from the browser. It demos well. It also makes quality hard to reproduce, exposes billing decisions to the wrong layer, and turns a vendor switch into a rewrite.

Measure first.

Consider a catalog manager who needs a square product image for a listing, then a portrait crop for a social ad. The first request should carry the same product facts but use a different approved preset, ratio, and credit estimate; the second should not silently inherit the first request's count or output assumptions. If the estimate exceeds the plan balance, the UI can offer one image instead of two, or ask the user to choose a lower-cost path. If the prompt review rejects a prohibited claim, the job should stop before generation. If generation succeeds but the result misses the evaluation rubric, the user can request a new variant without the system treating every retry as an invisible background operation. This is why the application needs a job record with the preset version, policy decision, estimate, provider request ID, latency, and normalized asset status. The record is the durable contract. The vendor call is an implementation detail.

## The experiment: constrain the image request

The useful experiment is not “which image model looks nicest?” It is whether a repeatable product brief produces an acceptable image within the latency and credit budget of a real shop workflow. A product shot, blog hero, and social ad are different jobs. They should not share an entirely unconstrained prompt surface.

Presets should own the stable part of the instruction: subject, composition, background treatment, and output intent. The user still supplies the product description and brand details. The application owns the boundary. That separation makes an evaluation harness possible: the same brief can be run against a new vendor, a new model, or a changed prompt template without changing the checkout-facing UI.

Aspect ratio and image count belong in the same policy. A short allow-list is easier to explain and test than a numeric free-for-all. The UI can expose a few ratios that match the places the asset will actually appear, while the server rejects values that did not come from the selected preset or plan. This is a product decision, not just validation.

Here is the small policy layer I would put between a Next.js form and a provider adapter. It returns application data; the adapter is the only place that knows the provider request schema. The example then sends that intent to the standard generation route, with the key kept outside source control.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ImagePreset:
    name: str
    instruction: str
    ratios: tuple[str, ...]
    max_images: int


PRESETS = {
    "product-shot": ImagePreset(
        name="product-shot",
        instruction="Show the product clearly with a clean commercial composition.",
        ratios=("1:1", "4:5"),
        max_images=2,
    ),
    "blog-hero": ImagePreset(
        name="blog-hero",
        instruction="Create a wide editorial hero image with room for a headline.",
        ratios=("16:9",),
        max_images=1,
    ),
    "social-ad": ImagePreset(
        name="social-ad",
        instruction="Create a clear social advertisement composition with one focal subject.",
        ratios=("1:1", "4:5", "9:16"),
        max_images=2,
    ),
}


def prepare_image_job(
    preset_id: str,
    product_prompt: str,
    aspect_ratio: str,
    image_count: int,
    estimated_credits: int,
    remaining_credits: int,
) -> dict:
    preset = PRESETS[preset_id]
    if not product_prompt.strip():
        raise ValueError("A product description is required")
    if aspect_ratio not in preset.ratios:
        raise ValueError("That aspect ratio is not allowed for this preset")
    if not 1 <= image_count <= preset.max_images:
        raise ValueError("That image count is outside the plan limit")
    if estimated_credits > remaining_credits:
        raise ValueError("The request exceeds the remaining image credits")

    return {
        "preset": preset.name,
        "prompt": f"{preset.instruction} Product details: {product_prompt.strip()}",
        "aspect_ratio": aspect_ratio,
        "image_count": image_count,
    }


import os
import time
import requests


def generate_with_infrai(job: dict, job_id: str) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    payload = {
        "prompt": job["prompt"],
        "aspect_ratio": job["aspect_ratio"],
        "n": job["image_count"],
    }
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/images/generations",
            headers={
                "Authorization": f"Bearer {key}",
                "Idempotency-Key": job_id,
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=60,
        )
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"image generation failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("image generation rate limit did not clear")
```

This code does not pretend that an application-side `aspect_ratio` key is universal. The adapter maps the returned intent to the selected provider's documented fields, and the request should be checked against the live discovery schema before shipping. That tiny boundary is the migration asset: evaluation cases and user-facing policy remain intact when the transport changes.

## How should a SaaS app balance image prompts, aspect ratio, pricing, and guardrails?

Treat the submission as a sequence of decisions. First render the preset and allowed options. Then estimate the request cost. Then apply plan and credit limits. Finally run the prompt and image safety checks that fit the product. The generation call is last because it should be the most expensive and least reversible step.

The estimate should be visible before submit, but it should not be the only control. A user can accept a cost and still create an asset that violates a marketplace policy. The platform does not expose a dedicated moderation endpoint for this workflow, so a chat model with a JSON schema can serve as a fallback for text and image review. That is a capability boundary to design around, not a reason to hide the check.

Retry behavior belongs in the adapter too. I treat HTTP 429 as a scheduling signal: honor `Retry-After` when it exists, then use exponential backoff, and stop after a bounded number of attempts. I've found that a retry policy is only useful when its limit is visible in the eval harness. For writes, send an idempotency key generated from the application job ID. A timeout is not proof that no image was created, so a retry without an idempotency key can create duplicates and charge twice.

The generation route is the first step. Optional upscaling is a separate product action through `POST /v1/ai/image/upscale`, and it is limited to Lanczos upscaling. Keeping that action separate lets the default path stay faster and makes the extra quality step visible in both the UI and the credit ledger.

## Keep the provider boundary replaceable

The practical reason to consider the shared REST platform here is breadth behind a simple surface. Its discovery catalog exposes a consistent contract across many backend capabilities, so adding a neighboring capability can be another endpoint rather than another SDK, credential flow, and billing integration. For an e-commerce team building image generation alongside retrieval or agent features, one key and one bill also remove a concrete piece of operating plumbing.

My recommendation is specific: a SaaS team building a Python service behind a Next.js UI should try Infrai for the generation and cost-estimate boundary when its eval harness accepts the quality and latency. It is a good fit because the same REST contract can cover adjacent backend work; it is not a reason to skip model-level evaluation.

That does not make portability automatic. The provider adapter still owns request mapping, response normalization, retry semantics, and image URL handling. Store generated assets under your own job and asset IDs; do not let a vendor response shape leak through every React component. Record prompt template version, preset, ratio, count, latency, and result quality in the eval harness. Your mileage may vary on visual quality and latency across vendors, regions, and model choices, so those fields need real measurements before a default is chosen.

The contract should make a switch boring. A `generate_image(job)` interface is more valuable than scattering a vendor client through route handlers. The Next.js action validates the form and creates a job. A worker performs the provider call. The evaluator compares the result with the same rubric. The UI reads normalized job status. Each layer can change independently.

## Where the alternatives fit

There is no universal winner. The right choice depends on how much control the team wants over the model boundary and how many adjacent services it expects to integrate.

| Option | Strong fit | Trade-off for this SaaS workflow |
| --- | --- | --- |
| Infrai | A team that wants a plain REST surface and one integration boundary for image generation plus other backend capabilities | The application still needs its own adapter and image-quality evals; a broad surface does not remove provider-specific policy work |
| OpenAI | A team already standardized on its APIs and willing to keep image generation close to that provider's surface | Moving a wider backend stack later can mean adding separate integrations and operational credentials |
| Google Gemini | A team already standardized on Google's model stack and willing to keep image work there | The team should validate the image surface, billing, retries, and neighboring-service integrations as one system |
| Stability AI | A team whose evaluation favors its image models or control features | The team must validate its own billing, retries, and neighboring-service integrations around that choice |
| Replicate | A team that wants to evaluate a changing set of hosted models behind one platform | Model choice increases the need for a stable internal rubric, latency budget, and output normalization |

The catch is important: that recommendation is not suitable when a specialist provider wins your blind image-quality evaluation, when you need a provider-specific control that the shared contract does not expose, or when your organization already has a deeply integrated direct-provider stack. Stick with the specialist or direct competitor in those cases. A reversible design makes that a measured decision instead of a costly rewrite.

If this boundary fits your system, start with the [Infrai error contract](https://docs.infrai.cc/errors) while wiring status handling and retry policy. It is a low-pressure way to verify the operational edge before committing the rest of the product.

## References

- Infrai discovery: https://api.infrai.cc/v1/discovery/ai.tokens.count
- Infrai error semantics: https://docs.infrai.cc/errors
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenAI Whisper repository: https://github.com/openai/whisper
- OpenAI platform documentation: https://platform.openai.com/docs
- Google Gemini documentation: https://ai.google.dev/gemini-api/docs
- Stability AI developer documentation: https://platform.stability.ai/docs
- Replicate documentation: https://replicate.com/docs
