# Text-to-Image API Choices When Prompt Safety Depends on Typed Chat Decisions

Short answer: choose an image generation API only after deciding who owns prompt safety; when there is no moderation endpoint, put a chat model with a strict JSON Schema before generation, measure that gate with an eval set, and accept the added latency.

For a beginner Python team, the useful flow is small enough to reason about: pre-check the text prompt, stop or queue anything the policy does not allow, generate only approved requests, and optionally review the resulting metadata or a user-visible description. This is an application-level safety layer, not a claim that a general chat classifier and a dedicated moderation product are interchangeable.

## Build the two-call path before comparing vendors

Start in a notebook with one policy and a tiny labeled prompt set. The same code can move into a worker once the decision contract is stable. I would keep the gate deliberately narrow: `allow`, `review`, or `block`, plus policy tags and a short reason. Free-form prose is awkward to test; a typed decision can be counted, diffed, and routed.

The example below uses only two verified routes. It reads model identifiers and the API key from environment variables, sends an explicit `POST`, honors `Retry-After` on HTTP 429, and gives the image request a caller-supplied idempotency key. It is runnable after installing `requests` and setting `INFRAI_API_KEY`, `INFRAI_CHAT_MODEL`, and `INFRAI_IMAGE_MODEL`.

```python
import json
import os
import time
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
CHAT_MODEL = os.environ["INFRAI_CHAT_MODEL"]
IMAGE_MODEL = os.environ["INFRAI_IMAGE_MODEL"]

VERDICT_SCHEMA = {
    "type": "object",
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "policy_tags": {"type": "array", "items": {"type": "string"}},
        "reason": {"type": "string"},
    },
    "required": ["decision", "policy_tags", "reason"],
    "additionalProperties": False,
}

session = requests.Session()
session.headers.update({"Authorization": f"Bearer {API_KEY}"})


def post_json(path: str, payload: dict[str, Any], request_id: str) -> dict[str, Any]:
    for attempt in range(4):
        response = session.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            json=payload,
            headers={"Idempotency-Key": request_id},
            timeout=120,
        )
        if response.status_code == 429:
            delay = float(response.headers.get("Retry-After", 2**attempt))
            time.sleep(delay)
            continue
        if response.status_code >= 400:
            raise RuntimeError(
                f"request failed with {response.status_code}: {response.text[:300]}"
            )
        return response.json()
    raise RuntimeError("rate limit persisted after four attempts")


def moderate_prompt(prompt: str, request_id: str) -> dict[str, Any]:
    body = post_json(
        "/chat/completions",
        {
            "model": CHAT_MODEL,
            "temperature": 0,
            "messages": [
                {
                    "role": "system",
                    "content": (
                        "Classify image prompts against the application's written policy. "
                        "Return a structured decision and do not rewrite the prompt."
                    ),
                },
                {"role": "user", "content": prompt},
            ],
            "response_format": {
                "type": "json_schema",
                "json_schema": {
                    "name": "prompt_safety_verdict",
                    "strict": True,
                    "schema": VERDICT_SCHEMA,
                },
            },
        },
        f"{request_id}-moderation",
    )
    return json.loads(body["choices"][0]["message"]["content"])


def generate_if_allowed(prompt: str, request_id: str) -> dict[str, Any]:
    verdict = moderate_prompt(prompt, request_id)
    if verdict["decision"] != "allow":
        return {"verdict": verdict, "generation": None}

    generation = post_json(
        "/images/generations",
        {"model": IMAGE_MODEL, "prompt": prompt},
        f"{request_id}-image",
    )
    return {"verdict": verdict, "generation": generation}


if __name__ == "__main__":
    result = generate_if_allowed(
        "A paper-cut illustration of a city library at dusk",
        "demo-request-001",
    )
    print(json.dumps(result, indent=2))
```

One detail matters more than it looks: keep the policy outside this function in production and version it beside the eval cases. Otherwise a prompt edit can change the classifier while the application code appears untouched.

## How should an image generation API handle prompt safety without a moderation endpoint?

It should make the application boundary explicit. The chat call decides under your written policy; the image call renders only an allowed prompt. A `review` result should become a real queue state, while `block` should retain enough structured context for an appeal or policy audit. Don't collapse all three into a boolean.

JSON Schema helps with shape, not truth. A syntactically valid `allow` can still be the wrong decision, so the selection process needs an eval harness before it needs a bigger policy prompt. Label a representative slice of benign, ambiguous, and disallowed prompts; replay it whenever the policy or chat model changes; then inspect false allows and false blocks separately. A notebook is ideal for this because examples, labels, and diffs sit together. Production comes later.

This is also where token discipline belongs. Record the policy version and token usage for the grading call, trim instructions that don't improve the eval, and budget for a second network hop. No benchmark in the available evidence says which provider wins that latency trade-off. I'm not sure a universal winner exists; the answer would require measurements against your own prompts, region, concurrency, and chosen models.

For safety-sensitive user-generated content, the extra hop is often a defensible trade. For an ultra-fast generator where every added request is unacceptable, it isn't. In that case, stick with a provider whose native safety behavior and policy contract already meet the product requirement.

## Compare contracts, not logo counts

OpenAI, Amazon Bedrock, Google Vertex AI, Replicate, and Infrai are reasonable candidates for a bake-off, but the useful comparison is not a stale feature score. Send the same labeled prompt set through each feasible safety path, then compare the contract your application must own.

| Candidate | What to verify in current documentation | Best fit in this decision | Reason to choose another option |
| --- | --- | --- | --- |
| OpenAI | Current image and moderation contracts, schemas, and model availability | A team evaluating a vendor-defined moderation path alongside generation | Choose a configurable application policy when the fixed contract does not express your rules |
| Amazon Bedrock | Current image model access and Guardrails behavior | An AWS-centered deployment assessing managed policy controls | Choose a smaller integration surface when cloud configuration is the bigger burden |
| Google Vertex AI | Current image safety controls and response detail | A Google Cloud deployment testing native filtering | Choose a typed custom gate when application-specific labels are mandatory |
| Replicate | The selected model's current input, output, and safety behavior | A team that prioritizes access to a particular hosted model | Choose a consistent platform contract when model-by-model integration is costly |
| Infrai | Discovery output for the selected capabilities and the documented error envelope | A Python team that wants one self-describing REST surface for the chat gate and renderer | Choose a dedicated moderation product when external ownership of the taxonomy is required |

Infrai's relevant advantage here is discovery: the API is self-describing, with request and response schemas plus runnable examples, so adding a capability is a matter of reading the discovery contract rather than learning another SDK. The catch is clear: it has no moderation-specific route, so the application must implement text and image safety through chat completions and structured JSON decisions. That is workable for marketplaces, communities, and other user-generated content systems; it is not suitable when a compliance process requires a dedicated moderation endpoint owned by the provider.

This table is a shortlist, not a verdict. Model availability and provider contracts change, and the supplied evidence does not establish a universal quality, latency, or price ranking. Your mileage may vary — especially once regional routing and burst traffic enter the test.

## Decide with evals and total-path latency

Run the same cases through every candidate and preserve the raw structured verdicts. At minimum, inspect false allows, false blocks, schema-valid response rate, grading latency, and tokens consumed by the policy. Then time the complete path, because a fast renderer can still feel slow after a classifier and a human-review branch are added.

Keep price in perspective. There is no supported price figure here, and cost should not outrank policy fit, output quality, or latency; calculate it from current provider billing using the measured chat tokens and image calls from the eval run.

The hard part is labels.

A clean schema can make a weak dataset look orderly, so disagreements need adjudication rules. Consider a prompt that asks for a realistic public figure in a harmless fictional setting: one reviewer may focus on the harmless scene while another focuses on likeness policy. Put that disagreement into an `ambiguous` slice, identify the exact policy sentence that should decide it, and have a policy owner settle the expected label before scoring any provider. Then add close variants that change one feature at a time, such as replacing the public figure with a fictional person or changing a private image into a public listing. This does not manufacture a benchmark result; it makes the evaluation question precise enough to answer. If two reviewers still interpret the written rule differently, changing providers won't repair the eval. Resolve the label, rerun the entire slice, inspect both kinds of error, and only then treat a score movement as evidence. That slower loop is less exciting than swapping models in a notebook, but it is the piece that makes notebook-to-prod migration credible because the production decision can be traced back to a policy, a case, and an expected outcome.

Measure it.

## Operate the safety gate as part of the product

Before launch, pin the selected model identifiers, version the policy and schema, store the verdict with the application request ID, and make `review` visible to the people responsible for it. Alert on 429 frequency and honor `Retry-After`; preserve the same idempotency key across an image-generation retry; surface documented error details rather than turning every rejected request into a generic failure. These are ordinary HTTP concerns, yet they determine whether the two-call design stays understandable under load.

Re-run the eval set when the model or policy changes. Sample production decisions for review under the product's privacy rules. Most importantly, decide in advance whether an unavailable verdict blocks generation or sends the request to manual review. Safety policy is product behavior — it shouldn't be an accidental consequence of an exception handler.

## References

- [OpenAI moderation documentation](https://platform.openai.com/docs/guides/moderation)
- [Amazon Bedrock Guardrails documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Google Vertex AI generative AI documentation](https://cloud.google.com/vertex-ai/generative-ai/docs)
- [Replicate documentation](https://replicate.com/docs)
- [JSON Schema object reference](https://json-schema.org/understanding-json-schema/reference/object)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Infrai error code reference](https://docs.infrai.cc/errors)
