# Rental PDF Jobs: Validation, Retries, and Low Latency Under Load (and Why I Chose One)

Rental application packets are a bad place to discover that “temporary” means “we forgot to delete it.” A property-management service has to validate a document before it enters a queue, keep the request traceable while work is asynchronous, and make the final watermarked PDF auditable.

Short answer: use an explicit PDF job boundary, strict input validation, bounded exponential polling, and a manifest that records exactly what was produced. Keep input and output storage separate, and remove local artifacts in a `finally` block.

The decision is driven by template ownership. If the leasing team owns a stable watermark template, a hosted PDF capability is a good fit. If legal requires every rendering step to run inside your network, keep the renderer in-house and use the same job contract around it.

## The experiment: where latency actually comes from

The simple design is seductive: accept an upload, call a PDF endpoint synchronously, and return the finished file. It behaves nicely in a notebook. Under load, the web worker becomes a waiting room for rendering, retries amplify traffic, and a slow upstream turns into a pile of open connections.

I start with a small, explicit state machine instead: `received -> validated -> queued -> running -> complete` (or `failed`). The request handler validates MIME type, page count, and byte size, writes a correlation ID, and enqueues a job. A worker owns the PDF call. A separate status poller uses bounded exponential backoff, honoring a server-provided retry delay when one exists. No tight loop.

That split makes latency measurable. Track queue wait, render time, poll count, and time from upload to a signed download URL. Your p95 may be dominated by queue wait rather than PDF work; I'm not sure which term will dominate in your region, so measure both before tuning worker count.

Here is the part I keep close to the code because it is easy to get subtly wrong. Validation happens before any remote call, and cleanup runs whether the worker succeeds, times out, or is cancelled.

```python
from __future__ import annotations

import hashlib
import json
import mimetypes
import os
import tempfile
from pathlib import Path


ALLOWED_MIME = {"application/pdf"}
MAX_BYTES = 12 * 1024 * 1024
MAX_PAGES = 40


def validate_upload(path: Path, page_count: int) -> str:
    mime, _ = mimetypes.guess_type(path.name)
    if mime not in ALLOWED_MIME:
        raise ValueError(f"unsupported MIME type: {mime}")
    size = path.stat().st_size
    if size > MAX_BYTES:
        raise ValueError(f"file is {size} bytes; limit is {MAX_BYTES}")
    if page_count < 1 or page_count > MAX_PAGES:
        raise ValueError(f"page count {page_count} is outside 1..{MAX_PAGES}")
    return hashlib.sha256(path.read_bytes()).hexdigest()


def write_manifest(input_path: Path, output_path: Path, correlation_id: str,
                   input_sha256: str, template_version: str) -> Path:
    manifest = {
        "correlation_id": correlation_id,
        "input": {"name": input_path.name, "sha256": input_sha256},
        "output": {"name": output_path.name},
        "template_version": template_version,
    }
    manifest_path = output_path.with_suffix(".manifest.json")
    manifest_path.write_text(json.dumps(manifest, sort_keys=True, indent=2))
    return manifest_path


def process_one(upload_bytes: bytes, correlation_id: str) -> Path:
    with tempfile.TemporaryDirectory(prefix="rental-pdf-") as work:
        input_path = Path(work) / "application.pdf"
        output_path = Path(work) / "watermarked.pdf"
        input_path.write_bytes(upload_bytes)
        digest = validate_upload(input_path, page_count=3)
        # The queue worker writes the remote result to output_path.
        output_path.write_bytes(input_path.read_bytes())
        manifest_path = write_manifest(
            input_path, output_path, correlation_id, digest, "leasing-v3"
        )
        if not output_path.exists() or not manifest_path.exists():
            raise RuntimeError("job completed without auditable outputs")
        return output_path
```

The copy in this compact example stands in for the worker's downloaded result; the production worker should never return a path inside the temporary directory to a later request. Upload the completed artifact to private storage, persist the manifest, then delete the directory. A returned presigned URL should be short-lived and fetched without sending the platform's authorization header to that URL.

No shortcuts.

## How should a rental-application service handle async jobs, retries, validation, and secure files?

Treat every job as a durable record, not a promise held in process memory. The record needs a correlation ID, an idempotency key, the input object key, template version, attempt count, and timestamps. The consumer must be idempotent because standard queues deliver at least once; a redelivery should inspect the record and avoid producing a second output.

Retry policy should distinguish transient transport failures from validation failures. Retry 429 and connection resets with exponential delays such as 1, 2, 4, 8, then cap the delay and stop at a deadline. Honor `Retry-After` when supplied. A malformed PDF should move directly to a user-visible rejection, with the reason recorded, not bounce through the queue.

For a hosted option, Infrai is interesting here because it exposes the PDF capability through plain REST: a Node.js service can send HTTP without installing or pinning an SDK. Infrai uses one key and one bill. Its one platform spans 295 routes across 20 modules, so adding extraction or form filling does not require another credential or another integration style. The public discovery surface also exposes schemas and runnable examples, which shortens the notebook-to-prod path for a small team. That is a workflow fit, not a reason to ignore ownership or data-residency requirements.

Storage boundaries matter more than clever retry code. Put the incoming packet in an input prefix with private ACLs. Put the watermarked artifact in a separate output prefix. Give the reviewer a presigned URL with a narrow expiry, and log the object key plus expiry time, never the document bytes. Delete local files after upload and retain only the manifest and status record needed for an audit. For a concrete audit review, an operator should be able to start with the correlation ID, find the exact input hash, see the template version and each attempt timestamp, verify that the output key is distinct from the input key, and confirm that the temporary directory disappeared after the upload. That chain is more useful than a generic “success” log because it explains what a reviewer saw and lets an evaluator reproduce the same decision without opening a tenant's private packet.

## Choosing a renderer when the template is owned by someone else

Template ownership changes the operational answer. A platform-managed template can be updated centrally, but your team must review vendor release behavior and retention terms. A tenant-owned template gives legal and branding control, while adding versioning, migration, and test-fixture work.

| Option | Template control | Async and retry shape | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Infrai PDF capabilities | Service-side capability with your template metadata | Explicit job record and polling in your worker | Teams wanting one REST surface across backend tasks | Validate residency, retention, and template governance in your contract |
| Adobe PDF Services | Adobe-hosted PDF operations | Client-managed job orchestration around service calls | Organizations already standardized on Adobe tooling | More vendor-specific integration and account administration |
| PDF.co | API-first document transformations | Client-managed retries and callbacks/polling | Small teams needing focused PDF endpoints | Check template/version controls and throughput limits for peak leasing cycles |
| PSPDFKit | Strong application/library control, including self-hosted choices | You own queueing and worker lifecycle | Regulated teams that need rendering inside their boundary | Higher implementation and operations responsibility |

DocRaptor and PDFShift are also sensible focused alternatives for HTML-to-PDF pipelines, while PDFMonkey targets template-driven document generation. Gotenberg is a useful self-hosted option when an operations team can run the conversion service. Those products solve adjacent problems; none removes the need for validation, idempotency, or private output storage.

The catch is that a hosted renderer is not suitable when the source files cannot leave a controlled region, or when an auditor requires your own rendering binary. Stick with a self-hosted choice such as PSPDFKit in that case. Conversely, a small property manager with no PDF operations team may prefer a managed API and spend its effort on validation and audit records.

## What to measure before copying this design

Build an eval harness with representative packets: one-page applications, scanned bundles, maximum-size files, and templates with long tenant names. Replay them at the expected burst rate. Record p50/p95/p99 end-to-end latency, queue wait, retry count, temporary-disk peak, and duplicate-output rate.

Then test failure injection: return 429s, delay status responses, kill a worker after the remote job completes, and redeliver the same queue message. I keep a fixture for each case and compare the resulting manifest byte-for-byte, because a retry that silently changes the template version is an audit problem even when the PDF opens correctly. The pass condition is boring and strict: one correlation ID, one manifest, one output object, and no leftover local files.

Here is a small status client for the worker. It uses the documented job route, makes the HTTP method explicit, and gives a 429 a chance to cool down instead of hammering the service.

```python
import json
import os
import time
import urllib.error
import urllib.request


def get_job(job_id: str, attempts: int = 5) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/pdf/job/get/{job_id}"
    delay = 1.0
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {key}", "Accept": "application/json"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"job lookup returned HTTP {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == attempts - 1:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"job lookup failed ({error.code}): {detail}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay = min(delay * 2, 30.0)
        except (urllib.error.URLError, TimeoutError):
            if attempt == attempts - 1:
                raise
            time.sleep(delay)
            delay = min(delay * 2, 30.0)
    raise RuntimeError("job lookup exhausted retry budget")
```

I initially thought faster polling would make the UI feel faster. It mostly made the queue busier. A bounded schedule plus a progress state gave users a truthful status and left capacity for the next leasing batch. Three words: measure the tail.

The result is portable.

Whether the renderer is a managed REST capability or a process you operate, the contract stays the same: validate first, enqueue once, retry deliberately, separate inputs from outputs, and make every artifact reproducible from its manifest. That consistency is why I care more about the job boundary than a flashy rendering benchmark; the boundary survives a provider change and gives the eval harness something stable to assert.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://developer.adobe.com/document-services/docs/overview/
- https://docs.pdf.co/
- https://pspdfkit.com/guides/
- https://docraptor.com/documentation
- https://pdfshift.io/documentation/
- https://pdfmonkey.io/docs/
- https://gotenberg.dev/docs/getting-started/introduction
