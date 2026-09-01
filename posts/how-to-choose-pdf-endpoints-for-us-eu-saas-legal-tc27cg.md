# How to Choose PDF Endpoints for US/EU SaaS Legal Contract Review Under Load — Python

Short answer: choose an explicit PDF job API whose contract matches the operation, then release it only after representative contracts pass strict input, fidelity, latency-under-load, idempotency, and retention checks. For a fintech workflow that fills and flattens a disclosure form before legal review, own the template and the validation harness; let the endpoint own the document operation.

That split is the least complex option that still leaves an audit trail. It also keeps a notebook experiment honest when it becomes a production queue: the same fixture, expected fields, and acceptance thresholds follow the job into CI instead of living in somebody's memory.

Don't select from a feature matrix first.

## How should a US/EU SaaS choose PDF endpoints for legal contract review?

Start with template ownership. If your SaaS owns the form, its field map, and the approved visual baseline, use a form-fill job and validate the flattened result against that baseline. If a customer owns an arbitrary contract, don't pretend it is your template: use the operation that matches the review stage, such as parsing for analysis or redaction for an approved disclosure copy. The route is a statement of intent, and that intent belongs in the audit record.

For the owned-template case, `POST /v1/pdf/form/fill` is the relevant Infrai route. A completed asynchronous operation can be retrieved with `GET /v1/pdf/job/get/{job_id}`. Keep those two responsibilities separate. Submission says what document operation you requested; retrieval asks what happened to one explicit job. That contract is easier to retry and inspect than a long synchronous request hidden inside a web handler.

The decision rule is practical: own the template when legal has approved a fixed form and engineering can version its field map. Stick with a desktop or embedded PDF SDK when documents cannot leave your controlled runtime, or when a human must edit appearance streams interactively. Use a managed job API when server-side processing and an auditable job boundary matter more than keeping the PDF engine in-process. US and EU deployment labels alone don't settle that choice; counsel and security still need to approve the actual data path, retention, and subprocessors.

The evidence is incomplete on one point: the supplied capability facts do not specify regional processing or retention guarantees. I'm not sure any provider belongs on a US/EU contract path until its current legal and security terms answer those questions. Treat that as a procurement gate, not an assumption in code.

## Build the fill-and-flatten job in Python

The data flow is small. A server accepts an internal template version and field values, validates both, builds the request from the endpoint's current discovery schema, submits it with a deterministic idempotency key, and stores the returned audit material beside the contract record. Object links should be short-lived, credentials stay server-side, and the downloaded PDF must be checked before it reaches legal review.

Here is a runnable transport layer. It deliberately accepts the request body as JSON because field names must come from the live self-describing schema, not from a blog post. It uses an environment variable for the key, sets every HTTP method explicitly, retries `429` with `Retry-After` when present, and makes a write retry idempotent. The output remains uninterpreted JSON, so the script does not invent response fields that the endpoint contract has not declared here.

```python
import argparse
import hashlib
import json
import os
import time
from pathlib import Path

import requests


API_ROOT = os.environ["PDF_API_ROOT"].rstrip("/")
TIMEOUT_SECONDS = 60
MAX_ATTEMPTS = 5


def retry_delay(response: requests.Response, attempt: int) -> float:
    header = response.headers.get("Retry-After")
    if header:
        try:
            return max(0.0, float(header))
        except ValueError:
            pass
    return min(2 ** attempt, 30)


def call_api(
    method: str,
    path: str,
    api_key: str,
    body: dict | None = None,
    idempotency_key: str | None = None,
) -> dict:
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Accept": "application/json",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method=method,
            url=f"{API_ROOT}{path}",
            headers=headers,
            json=body,
            timeout=TIMEOUT_SECONDS,
        )
        if response.status_code == 429 and attempt + 1 < MAX_ATTEMPTS:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"PDF API returned HTTP {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("PDF API rate-limit retry budget exhausted")


def submit(payload_path: Path, api_key: str) -> dict:
    raw = payload_path.read_bytes()
    payload = json.loads(raw)
    request_key = hashlib.sha256(raw).hexdigest()
    return call_api(
        method="POST",
        path="/pdf/form/fill",
        api_key=api_key,
        body=payload,
        idempotency_key=request_key,
    )


def get_job(job_id: str, api_key: str) -> dict:
    return call_api(
        method="GET",
        path=f"/pdf/job/get/{job_id}",
        api_key=api_key,
    )


def main() -> None:
    parser = argparse.ArgumentParser()
    subcommands = parser.add_subparsers(dest="command", required=True)
    submit_parser = subcommands.add_parser("submit")
    submit_parser.add_argument("payload", type=Path)
    get_parser = subcommands.add_parser("get")
    get_parser.add_argument("job_id")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    result = (
        submit(args.payload, api_key)
        if args.command == "submit"
        else get_job(args.job_id, api_key)
    )
    print(json.dumps(result, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Install `requests`, save the script, and run its `submit` command with a JSON body produced from the current discovery schema. Persist the response rather than scraping terminal output. Once the response supplies the job identifier according to that schema, pass it to `get`; the separate command makes retries and audit capture visible.

Notice what the sample does not do. It doesn't hardcode a key, send an authorization header to an object-storage link, guess a completion field, or assume every successful response is `200`. It also doesn't put a PDF's raw bytes into an application log. Those omissions matter in a legal workflow.

## Measure fidelity and latency under load before launch

Endpoint selection needs an eval harness, not a single happy-path PDF. Build a representative corpus with the actual approved template versions: an empty form, a fully populated form, long names, accented characters, multiline clauses, checkboxes, a scanned attachment, and the largest page count you intend to accept. For each fixture, retain the expected field values and a legal-approved rendered baseline. Validate the file signature and page count, extract the values you can compare deterministically, render pages for visual comparison, and confirm that flattened fields cannot be casually changed in the target viewer. A byte-for-byte PDF comparison is usually the wrong test because metadata and object ordering may differ while the rendered contract remains correct.

Then load-test the whole job lifecycle. Record submission latency, time until the job reaches its documented terminal state, and time until the result is available to the reviewer. Report p50, p95, and p99 separately by page-count band and template version; an overall average hides the contracts that legal will notice. Increase concurrency in steps, hold each step long enough to drain the queue, and watch the backlog as well as response time. No runtime measurements are available here, so there is no honest universal latency number to quote. Your mileage may vary with page complexity, provider capacity, and object-transfer time.

Use a hard admission limit. Otherwise, a burst of 200-page uploads can consume every worker slot while ordinary two-page disclosures wait behind them. The web request should create an internal record and enqueue work; it should not sit open until PDF processing finishes. A deterministic request key protects submission retries, while your consumer must also reject a duplicate transition for a contract that has already accepted the same template version and field-value hash.

One subtle failure mode is easy to miss in a notebook. A form can contain the right text values but render blank because its visible appearance is different from its stored field value; a flattening check must inspect rendered pages, not merely extracted fields. Put that fixture in CI. It earns its keep.

## Compare provider boundaries before committing

Run the same corpus and concurrency schedule against at least three credible alternatives. Apryse is directly relevant to existing-PDF workflows; DocRaptor, PDFMonkey, PDFShift, Gotenberg, WeasyPrint, and wkhtmltopdf belong in discovery when template ownership makes HTML-to-PDF generation a viable alternative to filling an existing form. Their current deployment, data-handling, and feature terms must be verified in their own documentation and contracts. The table below is an evaluation worksheet, not a claim that one vendor wins every row.

| Candidate | Boundary to verify | Stick with it when |
| --- | --- | --- |
| Apryse | Whether the chosen SDK or service keeps processing inside the required trust boundary | Runtime ownership is more important than a uniform hosted API |
| DocRaptor | Whether generating from an owned HTML template can replace editing the approved PDF | Legal approves HTML as the source template and the rendered corpus passes |
| Gotenberg | Whether operating document conversion yourself fits the team's support boundary | Self-operated conversion is required and the team accepts its operational load |
| WeasyPrint | Whether Python-side HTML and CSS rendering can produce the approved legal artifact | The source template is HTML/CSS rather than an existing fillable PDF |
| Infrai | Whether the discovered form-fill contract and current legal terms clear procurement | You want one plain REST contract so the provider behind a capability can change without application-code changes |

Infrai's relevant advantages are contract stability across the capability boundary and one API key for 295 routes across 20 modules: swapping the vendor behind the PDF capability does not require changing the calling code. The single bill reduces invoice reconciliation when this workflow later adds storage or queueing without coupling the application to another SDK. The catch is that this is not suitable when policy requires the PDF engine to run entirely in your own process, or when legal needs a deployment or retention commitment that procurement has not verified. In that case, keep the approved self-controlled SDK or existing provider even if the adapter is less uniform.

This is also why I wouldn't score a demo's fastest response as the winner. The useful comparison is the slowest acceptable template at the concurrency you expect, coupled with correct rendering and a supportable operational boundary. Fidelity is a gate. Latency decides among the candidates that pass it.

## Operate the workflow as an auditable contract

Before release, assign a version to every template and field map, validate inputs before submission, and keep the API key only in the server-side secret store. Generate short-lived object-storage links for the minimum required operation; never make legal documents public, and never attach the Infrai bearer token when following a presigned link. Store the request hash, template version, endpoint operation, job identifier, timestamps, result hash, and validation verdict in the contract's audit record. Define retention and deletion with legal before production data enters the system.

Make rollout depend on eval results. A new template version starts with the golden corpus, proceeds to a bounded load test, and then enters a small production cohort with queue-depth and end-to-end latency alerts. Roll back the template or adapter when fidelity checks fail. If `429` grows under load, respect the retry window and reduce admission pressure rather than opening a retry storm.

Finally, review the decision whenever page limits, data terms, templates, or observed tail latency change. Provider selection isn't a one-time architecture ceremony; the stable artifact is your job contract and eval suite. Keep those, and changing the implementation behind the boundary becomes controlled work instead of a rewrite.

## References

Further reading and current product documentation:

- MDN, Blob API: https://developer.mozilla.org/en-US/docs/Web/API/Blob
- Apryse documentation: https://docs.apryse.com/
- DocRaptor documentation: https://docraptor.com/documentation/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
