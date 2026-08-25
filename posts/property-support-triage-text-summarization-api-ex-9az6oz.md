# Property Support Triage: Text Summarization API Example with Chat JSON Output

Short answer: a Node.js text summarization API or a Python chat client should make the JSON contract the product boundary for property-management ticket triage, split long ticket threads before the model call, and reject anything that fails validation or an eval. A fluent summary is not a correct routing decision.

Keep it boring.

The useful pipeline is small: normalize a ticket, classify its urgency and queue, preserve the evidence, and validate the result before a human or workflow consumes it. A long tenant thread may contain a maintenance request, a billing dispute, and a safety detail several messages apart. Imagine a tenant reporting water through a bedroom ceiling, then adding a question about a fee in the same thread. If the reducer sees only the final fee question, it can produce perfectly valid JSON and still route a possible maintenance emergency to billing. That is why the summarizer must compress the thread without turning uncertainty into a fact, and why the application needs an explicit evidence field instead of trusting the tone of a generated paragraph.

## What should a Python text summarization API return for a long property ticket?

Start with a narrow object that downstream code can trust. For this job, I use \`category\`, \`priority\`, \`summary\`, \`evidence\`, and \`needs_human_review\`. \`priority\` is an enum rather than a free-form adjective, and \`evidence\` holds short quotations or faithful paraphrases tied to the source text. The application still owns the final policy decision; the model proposes a structured record.

Here is a complete small example. It uses Python's standard library, sends one chat-completions request through a configurable OpenAI-compatible endpoint, and treats malformed JSON as a failed job. The endpoint is configuration, so the same test harness can compare several backends without changing the triage code.

```python
import json
import os
from dataclasses import dataclass
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


ALLOWED_PRIORITIES = {"urgent", "normal", "low"}
REQUIRED_FIELDS = {
    "category",
    "priority",
    "summary",
    "evidence",
    "needs_human_review",
}


@dataclass(frozen=True)
class Triage:
    category: str
    priority: str
    summary: str
    evidence: list[str]
    needs_human_review: bool


def validate_triage(value: object) -> Triage:
    if not isinstance(value, dict) or set(value) != REQUIRED_FIELDS:
        raise ValueError("unexpected triage fields")
    if not isinstance(value["category"], str) or not value["category"]:
        raise ValueError("category must be a non-empty string")
    if value["priority"] not in ALLOWED_PRIORITIES:
        raise ValueError("priority is outside the application enum")
    if not isinstance(value["summary"], str) or not value["summary"]:
        raise ValueError("summary must be a non-empty string")
    if not isinstance(value["evidence"], list) or not all(
        isinstance(item, str) for item in value["evidence"]
    ):
        raise ValueError("evidence must be an array of strings")
    if not isinstance(value["needs_human_review"], bool):
        raise ValueError("needs_human_review must be boolean")
    return Triage(**value)


def triage_ticket(ticket: str) -> Triage:
    base_url = os.environ["AI_BASE_URL"].rstrip("/")
    api_key = os.environ["AI_API_KEY"]
    model = os.environ["AI_MODEL"]
    prompt = (
        "Return exactly one JSON object with these fields: category (string), "
        "priority (urgent|normal|low), summary (string), evidence (array of "
        "strings), and needs_human_review (boolean). Do not invent facts. "
        "Set needs_human_review true for ambiguity, possible safety risk, or "
        "a request that needs a property manager's policy decision.\n\n"
        f"Customer-support ticket:\n{ticket}"
    )
    body = {
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0,
    }
    request = Request(
        f"{base_url}/v1/chat/completions",
        data=json.dumps(body).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        method="POST",
    )
    try:
        with urlopen(request, timeout=30) as response:
            payload = json.load(response)
    except (HTTPError, URLError, TimeoutError) as error:
        raise RuntimeError("triage request did not complete") from error

    content = payload["choices"][0]["message"]["content"]
    return validate_triage(json.loads(content))


if __name__ == "__main__":
    sample = (
        "Tenant in unit 4B says water is coming through the bedroom ceiling "
        "after rain. They also ask why last month's fee changed."
    )
    print(json.dumps(triage_ticket(sample).__dict__, indent=2))
```

The important boundary is \`validate_triage\`, not the prompt wording. If a response contains an extra field, an unknown priority, or evidence in the wrong type, the worker should mark the ticket for review and retain the raw response in a restricted log. It should not silently coerce "critical" to "urgent" or store a prose blob where a queue expects an enum.

## How does a Python summarization API keep chat JSON output correct for long text?

First, separate transport limits from semantic limits. A ticket thread can be too large for one request, but blindly slicing characters can separate a tenant's claim from the manager's reply. Split on message boundaries, keep a stable sequence number, and ask each chunk for the same contract. The final pass receives the ordered, validated chunk objects rather than the entire raw thread.

The reducer needs its own rules. It must merge evidence, preserve the highest supported priority, and set \`needs_human_review\` when chunks disagree. A final model call can write the prose summary, but application code should still enforce the enum and boolean. This is where notebook-to-prod work usually becomes real engineering: the notebook displays a plausible answer; the service must explain why a ticket entered a queue.

For a long thread, I would keep the chunk prompt focused on facts that affect triage:

```python
def chunk_messages(messages: list[str], max_chars: int = 12000) -> list[str]:
    chunks: list[str] = []
    current: list[str] = []
    size = 0
    for message in messages:
        if current and size + len(message) + 1 > max_chars:
            chunks.append("\n".join(current))
            current = []
            size = 0
        current.append(message)
        size += len(message) + 1
    if current:
        chunks.append("\n".join(current))
    return chunks
```

Character limits are only a conservative guard, not a token measurement. Your mileage may vary across languages, formatting, and model tokenizers; measure representative tickets before selecting a production threshold. I would leave room for instructions and the JSON response, then test the boundary with the longest real message in the fixture set.

## Which failure modes should the eval harness catch?

Schema validity is the first test, not the finish line. A ticket about a sparking outlet can produce valid JSON with \`priority: "low"\`; a thread can mention a rent balance only to quote an old message; a chunk reducer can duplicate evidence. Each case passes parsing while failing the job.

Build a frozen fixture set with labeled routing facts. Include a safety-related maintenance report, a billing-only question, a mixed-intent thread, an empty or nearly empty message, and a long conversation whose decisive detail appears near the end. Score required-fact recall, priority accuracy, unsupported claims, evidence fidelity, human-review recall, and duplicate evidence. Keep the expected fields in application-owned fixtures rather than in an ad hoc spreadsheet.

I have seen the most useful debugging signal come from saving each chunk's sequence number, prompt version, parsed object, and validation result. One concrete trap is a \`429\`: if retry handling converts a missing response into an empty summary, the ticket may look successfully triaged. Retry boundedly, record the attempt, and fail closed after the limit. A failed triage is visible; a confident wrong queue is not.

## What are the trade-offs of chat JSON output versus rules for ticket triage?

Rules should own hard safety and routing constraints. If the text contains a policy-defined emergency phrase, a deterministic guard can force human review before any model output is trusted. The model is useful for extracting a concise summary and evidence from messy conversation, but it should not become the only enforcement layer.

| Design | Strength | Limitation |
| --- | --- | --- |
| Rules only | Predictable decisions for explicit patterns | Brittle with indirect, multilingual, or multi-message requests |
| Model JSON only | Handles varied phrasing and returns a compact record | Can be validly formatted and still factually wrong |
| Hybrid triage | Rules guard high-risk cases while the model extracts context | Requires two test suites and a clear precedence rule |

The catch is operational ownership. A hybrid design is not suitable when the team cannot maintain policy rules and labeled eval fixtures; in that case, start with a human-reviewed extraction workflow and keep automation advisory. Stick with rules for regulated or safety-critical decisions whose conditions can be expressed precisely. The right question is not whether the summary sounds good. It is whether the resulting decision can be checked.

## A practical notebook-to-production checklist

Before release, pin the JSON field set and allowed values in code, then test missing fields, extra fields, wrong types, invalid enum values, empty content, and contradictory chunks. Exercise timeout and \`429\` behavior without treating either as a valid summary. Measure prompt and completion tokens beside quality scores, because a shorter request that drops the evidence is not an optimization.

Deploy the worker with an idempotency key derived from the ticket revision, so a retry cannot write two triage records. Keep raw customer text access-controlled, redact it from ordinary logs, and attach a trace identifier to the request, validation decision, and queue update. Review a sample of accepted and human-reviewed tickets after each prompt or model change. Small checks compound.

## References

- https://platform.openai.com/docs/guides/batch
- https://www.promptingguide.ai
