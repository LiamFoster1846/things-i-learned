# Node.js Structured Extraction Failures: Token Limits, Timeouts, and Evidence Recall

Short answer: a long-document text-to-JSON timeout is rarely fixed by raising one timer. Bound each unit of work by tokens and a shared deadline, preserve source evidence through chunking, and add embeddings or reranking only when a field-level eval proves that exhaustive extraction is the wrong trade-off.

The important output isn't merely valid JSON. It is valid JSON whose values can be traced to the input, reproduced under the same schema and prompt versions, and rejected when the evidence is absent. That changes the design from one heroic model call into a small data pipeline with observable failure states.

## How should Node.js long-document extraction handle token limits and timeouts?

Start by refusing to label every slow or failed job a timeout. There are at least four distinct conditions: the input cannot fit the selected context budget; the job exhausts an application deadline; retrieval omits the passage needed for a field; or generation returns an object that fails schema or evidence validation. They need separate counters because they need separate remedies. A larger request timeout cannot restore omitted evidence, and a smaller chunk cannot correct a bad reducer.

I would keep the Node.js request layer responsible for admission control, cancellation, and job status, while putting the extraction contract behind a language-neutral queue message. That message should carry a document version, schema version, prompt version, chunk or candidate IDs, an absolute deadline, and an idempotency key. A Python worker can then run the model-facing experiment without making HTTP connection lifetime part of the experiment. This is a practical notebook-to-prod boundary: the notebook and worker share fixtures and extraction logic; the web process does not become a second orchestration engine.

Budget output tokens before calculating the input allowance. Instructions, the schema, candidate text, and the expected response all compete for the same context window, so “the document fits” is the wrong test. The useful invariant is that the assembled request fits with a deliberate reserve. Measure it with the tokenizer used by the tested model path; character counts are only a rough proxy and can fail differently across tables, code, OCR output, and languages.

Chunk on document structure first. Headings, paragraphs, table rows, and record boundaries retain more meaning than a blind fixed-width slice. Use a token ceiling as the final guard, then assign stable source offsets before any normalization that changes positions. Overlap may rescue a sentence split across two chunks, but it also duplicates prompt tokens and can produce conflicting claims. Treat overlap as an eval parameter, not a magic constant.

Short inputs should stay simple.

If a document and its schema fit comfortably within the measured budget and meet the latency target, one request has fewer merge states and fewer opportunities to lose evidence. Chunking is a response to a demonstrated constraint, not a badge of production readiness.

## Make evidence recall the first experiment

The simplest failed approach is often “retrieve the top few chunks, ask for the whole schema, parse the JSON.” It looks efficient, yet it mixes three questions: did retrieval surface the evidence, did generation interpret it, and did the reducer preserve it? A final exact-match score cannot identify which stage caused a miss.

Freeze a representative eval set with expected values and source spans. For every schema field, record whether the answer-bearing span entered the candidate set before evaluating generation. That gives retrieval recall independently of extraction accuracy. Then record unsupported-value rate, schema-valid rate, conflict rate, input and output tokens per accepted document, and wall-time percentiles. Prompt cost belongs beside quality and tail latency; a low-token attempt that is rejected and repeated is not low cost in the completed workflow.

Embeddings help when a field has distinctive local evidence and exhaustive processing consumes too much of the budget. They are a lossy router, though. A top-k result is not a completeness guarantee. Fields assembled across distant passages, low-frequency exceptions, totals spanning tables, and clauses whose meaning depends on surrounding definitions may require exhaustive coverage or a field-specific retrieval plan.

Reranking has a narrower job: reorder a candidate set that already contains the relevant passage. Add it only if the pre-rerank recall is acceptable and the useful chunk regularly falls below the extraction cutoff. If evidence never enters the candidate pool, reranking cannot recover it. I'm not sure any universal candidate count is defensible without the corpus, tokenizer, schema, and latency target; a sweep over those values on frozen examples is what resolves the uncertainty.

Measure first.

This table is the decision record I want before changing the architecture:

| Observed eval result | Smallest sensible change | New failure to measure |
|---|---|---|
| Whole request exceeds the measured input allowance | Structure-aware chunking | Boundary omissions and duplicate evidence |
| Exhaustive chunks pass quality but miss the job deadline | Parallel bounded workers or field routing | Deadline exhaustion and cancellation lag |
| Retrieval has weak evidence recall | Improve segmentation, queries, or candidate breadth | Prompt growth and false candidates |
| Candidates contain evidence but ordering is poor | Rerank before extraction | Relevant passages dropped by reranking |
| JSON parses but values lack support | Evidence-aware validation | False rejection from offset drift |

Notice the order. Don't pay for an extra model stage until the preceding measurement names the failure it can fix.

## Build a deadline-aware, evidence-carrying core

The focused Python example below models the part worth sharing between a notebook eval and a production worker. It accepts already prepared chunks, stops admitting work after a monotonic deadline, validates each partial result, retains evidence spans, and refuses silent conflicts. The model adapter and tokenizer stay outside the example because their exact behavior must match the deployment path.

```python
from __future__ import annotations

from dataclasses import dataclass
from time import monotonic
from typing import Any, Callable, Protocol


@dataclass(frozen=True)
class Chunk:
    chunk_id: str
    start: int
    end: int
    text: str


@dataclass(frozen=True)
class Claim:
    field: str
    value: Any
    chunk_id: str
    evidence_start: int
    evidence_end: int


class Extractor(Protocol):
    def extract(
        self,
        chunk: Chunk,
        schema: dict[str, Any],
        deadline: float,
    ) -> list[Claim]: ...


def extract_with_evidence(
    chunks: list[Chunk],
    schema: dict[str, Any],
    extractor: Extractor,
    validate_claim: Callable[[Claim, Chunk], None],
    timeout_seconds: float,
) -> dict[str, Any]:
    if timeout_seconds <= 0:
        raise ValueError("timeout_seconds must be positive")

    deadline = monotonic() + timeout_seconds
    values: dict[str, Any] = {}
    evidence: dict[str, list[dict[str, int | str]]] = {}

    for chunk in chunks:
        if monotonic() >= deadline:
            raise TimeoutError("document deadline exhausted before next chunk")

        for claim in extractor.extract(chunk, schema, deadline):
            validate_claim(claim, chunk)
            if claim.field in values and values[claim.field] != claim.value:
                raise ValueError(f"conflicting evidence for {claim.field!r}")

            values[claim.field] = claim.value
            evidence.setdefault(claim.field, []).append(
                {
                    "chunk_id": claim.chunk_id,
                    "start": claim.evidence_start,
                    "end": claim.evidence_end,
                }
            )

    return {"data": values, "evidence": evidence}
```

This deliberately does not merge arrays, vote among values, or convert missing fields to `null`. Those are domain decisions. “The source explicitly states none,” “no supporting passage was found,” and “this field was not requested from this chunk” are different states; flattening them early creates plausible output that no longer describes the source. Consider a renewal date mentioned in a summary and then qualified in an amendment near the end of the document. Last-write-wins depends on processing order, majority voting may favor repeated stale text, and concatenation produces an invalid business value. The reducer needs a rule grounded in document structure, such as amendment precedence, plus evidence for both claims and an explicit conflict state when that precedence cannot be established. Lists need similar care: repeated entities might be duplicates, separate events, or references to the same event under different names. Normalize only what the domain contract permits, retain the contributing spans, and evaluate the reducer on conflicts rather than only on clean examples. A production reducer should expose unresolved disagreement for review instead of making a tidy guess.

There is another sharp edge: offsets must refer to a declared representation. If extraction sees normalized text while the reviewer opens the original bytes, an apparently precise span may point at the wrong phrase. Store the normalized artifact or maintain an offset map. Test this with whitespace changes, Unicode normalization, page headers, and table serialization as part of the ingestion suite.

The deadline is absolute and monotonic on purpose. Passing a fresh relative timeout to every chunk lets a document exceed its total budget one request at a time. The adapter should derive its remaining allowance from the deadline and propagate cancellation. A queue worker should also stop launching new calls once completion inside the remaining window is unreasonable. Fast failure is useful here — it preserves capacity for jobs that can still finish.

## Retry semantics and deployment boundaries

Retries are a protocol decision, not an exception handler around the entire document. RFC 9110 defines idempotent methods and the conditions under which a client can automatically retry after a communication failure. Many extraction submissions use a non-idempotent HTTP method, so automatic replay is unsafe unless the application defines deduplication. Derive an idempotency key from stable inputs such as document version, schema version, prompt version, and unit-of-work ID; persist the accepted result against that key. Sending a header alone does not create the guarantee.

Retry the smallest unit whose outcome is unknown, with bounded backoff and enough deadline remaining to finish. Do not replay schema-invalid output with the identical request and call that recovery. That is either a constrained repair attempt, a prompt or schema defect to evaluate, or a terminal validation result. Keep transport attempts separate from semantic attempts in telemetry.

For deployment, log identifiers and measurements rather than document contents or credentials: configuration fingerprint, model-path identifier, tokenizer version, schema and prompt versions, candidate IDs, token counts, remaining deadline, validation result, and retry reason. A configuration mismatch can resemble a model or chunking problem, so compare the normalized startup configuration between notebook, staging, and worker before tuning prompts. A 401 belongs in the authentication bucket; it should not inflate a generic timeout chart.

The catch is that this architecture is not suitable for every input. Use a deterministic parser when stable labels or grammar already define the source. Stick with exhaustive chunk processing when missing a low-similarity passage is worse than spending more tokens. Prefer an asynchronous batch job when legitimate processing time exceeds an interactive HTTP budget. And keep the whole-document path for small, uniform inputs that pass the eval; queues, embeddings, and rerankers would add operational surface without evidence of benefit.

Before copying the design, measure field-level accuracy, evidence recall before generation, unsupported-value rate, merge conflicts, p50 and p95 wall time, deadline exhaustion by stage, retries per accepted document, and tokens per accepted field. Replay the frozen set whenever the schema, prompt, tokenizer, segmentation, retrieval, reranker, or model path changes. Valid JSON is the starting line. Grounded, reproducible JSON is the result.

## Further reading

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- LiteLLM, open-source LLM gateway: https://github.com/BerriAI/litellm
