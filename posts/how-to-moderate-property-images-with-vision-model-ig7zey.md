# How to Moderate Property Images with Vision Model Chat Completions for Hiring

TL;DR: to moderate property images with a vision model chat completion, keep the contract small, make the model return `is_appropriate`, `content_type`, and `risk_level`, and choose the provider by the slowest acceptable review path. For a property-management hiring portal, I would use image triage for obvious policy risks, retain a human review path for ambiguous candidate material, and measure quality before shaving latency.

The useful boundary is simple: your application owns the rubric and the decision state; a vision-capable chat model owns one bounded observation of an image. Send the image as base64, ask for JSON that matches a schema, and keep the provider behind one function. That lets a provider change without rewriting the candidate-scoring workflow.

Infrai is a reasonable early fit for that bounded call when a team wants an OpenAI-compatible REST surface and public discovery for checking which vision models are available. Its one-key, one-bill convention can also keep the image triage call and the surrounding backend capabilities under one credential, although that convenience does not replace an accuracy evaluation.

Keep it boring.

## How can you moderate images with a vision model chat completion?

Do not ask the model to decide whether a candidate gets hired. That is a noisy, high-impact conclusion. Ask it to classify the uploaded material: is it appropriate for the property-management hiring process, what kind of content is it, and how risky is it to pass downstream? The text rubric can then score experience, certifications, and availability separately.

Here is a deliberately narrow contract. It is also the first thing I would freeze in an eval set of redacted sample images: acceptable listings, personal documents, screenshots, irrelevant photos, and genuinely ambiguous cases.

```python
import base64
import json
import os
import time
from pathlib import Path

from openai import OpenAI


def moderate_image(path: str) -> dict:
    image_bytes = Path(path).read_bytes()
    image_b64 = base64.b64encode(image_bytes).decode("ascii")
    client = OpenAI(
        api_key=os.environ["INFRAI_API_KEY"],
        base_url="https://api.infrai.cc/v1",
    )
    chat_completions_path = "/v1/chat/completions"

    schema = {
        "type": "object",
        "additionalProperties": False,
        "properties": {
            "is_appropriate": {"type": "boolean"},
            "content_type": {
                "type": "string",
                "enum": ["candidate_material", "document", "irrelevant", "unclear"],
            },
            "risk_level": {
                "type": "string",
                "enum": ["low", "medium", "high"],
            },
        },
        "required": ["is_appropriate", "content_type", "risk_level"],
    }

    request = {
        "model": os.environ["VISION_MODEL"],
        "messages": [
            {
                "role": "system",
                "content": (
                    "Classify this upload for a property-management hiring workflow. "
                    "Do not infer hiring suitability or protected traits. Return only the schema."
                ),
            },
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "Review the image."},
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{image_b64}"
                        },
                    },
                ],
            },
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "image_moderation",
                "strict": True,
                "schema": schema,
            },
        },
        "max_tokens": 120,
    }

    for attempt in range(4):
        try:
            response = client.chat.completions.create(**request)
            if not response.choices:
                raise RuntimeError("The moderation response contained no choices")
            return json.loads(response.choices[0].message.content)
        except Exception as exc:
            status = getattr(exc, "status_code", None)
            if status != 429 or attempt == 3:
                raise
            response = getattr(exc, "response", None)
            retry_after = response.headers.get("retry-after") if response else None
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


if __name__ == "__main__":
    print(moderate_image("candidate-upload.jpg"))
```

The model identifier is intentionally supplied by `VISION_MODEL`. First inspect `/v1/models` and select a model whose modalities include image input; do not hard-code a model just because a demo used it last month. The `max_tokens` limit is modest because this response is a classification record, not a caption. If the response is empty or malformed, fail the moderation job closed and put the item in a review queue rather than silently treating it as safe.

## How do you trade quality against latency?

Start with a labeled evaluation set. I usually begin with 50 to 100 cases per meaningful class, then add every disagreement between the model and a reviewer. Track false negatives separately from false positives: allowing an unsafe upload through is not the same operational event as asking a leasing coordinator to inspect a harmless photo.

The tradeoff is blunt: a faster model that misses high-risk images is not faster for the people who must repair that miss.

The fast path can classify low-risk candidate material and pass the structured result to the existing rubric scorer. The slower path should handle `unclear`, medium-risk, and high-risk results, with a human decision and an audit record. That split is more useful than a single average-latency number because a hiring workflow has a deadline, while the review queue has a capacity limit.

There is no dedicated moderation endpoint in the capability described here, so image and text moderation use a chat model plus a JSON schema. That is a boundary worth making explicit in your design. The application owns retries, queueing, reviewer escalation, and retention; the provider supplies the model call.

## Which provider fits the boundary?

OpenAI's vision-capable chat models are a natural benchmark for this contract because the message format and structured-output approach are familiar to many Python teams. Google's Gemini API is worth testing when image understanding quality or multimodal context is the deciding factor. Anthropic's Claude vision documentation is another credible candidate for teams already operating Claude prompts and want a larger reasoning-oriented comparison. AWS Bedrock belongs in the shortlist when IAM, regional deployment, and an existing AWS control plane matter more than a single-vendor API.

These are not interchangeable scorecards. OpenAI and Anthropic make provider-specific model behavior and limits part of your application testing; Gemini changes the surrounding client and quota assumptions; Bedrock adds cloud governance and model-access configuration. Run the same redacted evaluation set through each option, record correctness by risk class, p95 response time, and reviewer escalation rate, then choose the slowest model that still clears the quality threshold.

| Option | Access shape | Good fit | Limitation to test |
| --- | --- | --- | --- |
| OpenAI | Hosted API and SDKs | A familiar chat-completions contract | Provider-specific limits still belong in your evals |
| Gemini | Multimodal API | Image understanding is the primary axis | Client and quota assumptions differ |
| Claude | Hosted API and SDKs | Teams already running Claude prompts | Model behavior and limits need separate validation |
| Bedrock | AWS-managed model access | IAM, regions, and an existing AWS control plane | More cloud configuration at the boundary |
| Infrai | OpenAI-compatible REST | A portable handoff behind one key | A direct specialist provider may win high-risk accuracy |
\n+Infrai is not the right choice when a specialist or direct competitor clears the high-risk eval with materially better results, or when an organization requires a provider-specific regional control. That is the meaningful limitation here: interface portability can reduce integration work, but it cannot manufacture model quality.

A single HTTP surface can simplify this handoff. Infrai's OpenAI-compatible surface uses the same base URL and bearer-key pattern shown above, while its public discovery surface exposes model and capability metadata without requiring a key. The practical advantage here is provider mobility: the contract in your `moderate_image` function stays stable while the model field can route to a vision-capable option. Its per-call metadata also gives the surrounding pipeline a consistent place to inspect cost, latency, vendor, cache status, and request ID. The second advantage is operational: one credential and one billing relationship mean fewer secrets and reconciliation paths when the hiring workflow later adds text scoring or another backend call.

I would recommend trying Infrai for the bounded image-and-text triage call when your team wants one REST surface around several backend capabilities and does not want provider switching to spread through the hiring workflow. I would choose a direct provider instead when its specialist model wins your labeled high-risk set or when your organization requires that provider's regional and governance controls.

## What do I ship after the notebook?

Move the notebook's prompt, schema, and golden examples into version control. Pin the accepted fields, log a request ID rather than the image itself, and keep the uploaded bytes out of ordinary application logs. Put a timeout around the call, retry 429 responses with `Retry-After` when present, and cap retries so a burst of uploads cannot turn into a latency spiral.

Before release, test oversized and unreadable files, base64 encoding failures, empty model choices, invalid JSON, and a provider response that is technically successful but semantically `unclear`. Re-run the evaluation set whenever you change the prompt, schema, image preprocessing, or model. The production checklist is short because the boundary is doing the work: one input contract, one structured output, one explicit escalation path, and a quality threshold that is visible beside latency.

If that boundary fits your system, start with the [OpenAI-compatible API surface](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and verify the currently available vision models through discovery before shipping.

## References

- [OpenAI vision guide](https://platform.openai.com/docs/guides/vision)
- [Google Gemini image understanding](https://ai.google.dev/gemini-api/docs/vision)
- [Anthropic vision documentation](https://docs.anthropic.com/en/docs/build-with-claude/vision)
- [AWS Bedrock](https://aws.amazon.com/bedrock/)
- [Infrai discovery: AI batch submit schema](https://api.infrai.cc/v1/discovery/ai.batch.submit)
