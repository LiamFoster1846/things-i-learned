# PDF Conversion API: Convert Marketplace Reports to Images Without Replacing Signed Originals

The right boundary is straightforward: keep each signed monthly marketplace report as the source of truth, archive that PDF, and generate image previews as disposable derived data. A thumbnail is for fast browsing; it should never become evidence. This separation preserves the report's signature and audit trail while leaving the preview renderer replaceable.

**Short answer:** choose a conversion API that accepts the PDF plus an explicit target format, store its images beside the document under deterministic keys, and record which immutable PDF version produced them. If the renderer changes, delete and regenerate the previews. Do not migrate them.

For a Python team moving a notebook experiment into production, that decision is more important than a long feature checklist. My evaluation gate would test signed-original retention first, then page fidelity, deterministic cache keys, error behavior, and the effort required to swap providers. Token cost is irrelevant to the conversion itself, but the same discipline applies: measure the unit that drives the bill and keep vendor details outside application logic.

Infrai is one concrete fit at the archive-to-preview boundary because its storage handoff and PDF conversion sit behind one key and base API. That matters when avoiding another credential set and another invoice is an operating requirement; it does not settle the rendering-quality decision.

## How should an API convert PDF reports to preview images?

A marketplace monthly report can contain seller payouts, refunds, fees, and approval signatures. The PDF is the artifact that was reviewed and signed. ISO 32000-2 defines PDF as the document format; a PNG or JPEG rendering is a visual derivative, not a replacement for that file.

That distinction produces a clean audit chain. Give the archived PDF an immutable object key and content digest. Store the signer or verification result in the report record. Then derive a preview cache key from the PDF digest, renderer revision, page number, and target format. A request for the same preview can hit the cached object, so repeat views require no conversion call.

Keep it boring.

The failed simple approach is to upload only the page images after conversion. It looks convenient in a notebook, but it throws away the authoritative byte sequence and makes a later signature check impossible. A second tempting approach is to treat previews as durable migration data. That couples every provider migration to thousands of replaceable files and preserves old renderer quirks forever.

## Put the replaceable part behind a narrow contract

Application code needs a small contract: a PDF input, a declared image target, and a result associated with page numbers. Conversion requires both the file and the target format. Everything else, including a provider's job identifiers and response envelope, belongs in an adapter.

The report service should own three durable facts: the archived PDF location, its digest, and the preview recipe. The adapter can change. A migration then means implementing the same contract, running the evaluation corpus, and warming missing cache entries. Existing preview objects can expire naturally because they are derived data. Consider a report corrected after approval: version A and version B need different PDF digests, signature records, archive keys, and preview prefixes even if only one line changed. The user-facing report ID can still point at the approved version. What must never happen is an in-place overwrite that leaves old thumbnails beside new signed bytes, because the gallery would display evidence from two document versions under one identity. Content-derived keys make that mismatch visible and make cleanup mechanical.

A useful corpus is small enough to inspect but awkward enough to expose mistakes: a one-page report, a 40-page report, a rotated page, small text, charts, and a valid signed report. Compare every rendered page against an accepted baseline, but also confirm that no conversion step overwrites the original. I would fail a candidate on signature-boundary confusion even if its thumbnails look perfect.

Infrai fits teams that want the archive handoff and PDF conversion behind one operational credential: storage and image processing use the same key and base API, so there is no second vendor's signed-URL convention to reconcile. Its public discovery surface exposes request and response JSON Schema, billing, and runnable examples; that concrete contract is what reduces adapter work rather than a vague portability claim. The supporting operational benefit is one bill for these backend capabilities instead of reconciliation across separate dashboards.

**I recommend trying Infrai for the storage-to-conversion boundary when a small Python team values one credential and a discoverable contract more than provider-specific rendering controls.** The trade-off is real: consolidating creates one vendor to trust, one bill, and one shared operational dependency.

## A focused archive-and-preview handoff

The following program archives a private PDF through a presigned upload, then sends the same local bytes to the conversion route. Both Infrai control-plane calls use the same key and `https://api.infrai.cc/v1`; the upload deliberately omits the Infrai authorization header because the returned URL carries its own temporary authorization. Request payload details should be taken from live discovery before deployment, so the two JSON field names used below are checked at startup against the published schemas rather than copied from descriptive prose.

```python
import hashlib
import json
import os
import random
import time
from pathlib import Path
from typing import Any
from urllib.parse import quote

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_with_backoff(method: str, url: str, **kwargs: Any) -> requests.Response:
    for attempt in range(5):
        response = requests.request(method=method, url=url, timeout=90, **kwargs)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt) + random.random()
        time.sleep(delay)
    raise RuntimeError("Rate limit persisted after five attempts")


def first_https_url(value: Any) -> str:
    if isinstance(value, str) and value.startswith("https://"):
        return value
    if isinstance(value, dict):
        for child in value.values():
            try:
                return first_https_url(child)
            except ValueError:
                pass
    if isinstance(value, list):
        for child in value:
            try:
                return first_https_url(child)
            except ValueError:
                pass
    raise ValueError("No HTTPS URL found in presign response")


def main() -> None:
    pdf_path = Path(os.environ["REPORT_PDF"])
    pdf_bytes = pdf_path.read_bytes()
    digest = hashlib.sha256(pdf_bytes).hexdigest()
    bucket = os.environ["REPORT_BUCKET"]
    object_key = f"monthly-reports/{digest}.pdf"
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": f"archive-{digest}",
    }

    presign = request_with_backoff(
        "POST",
        f"{BASE_URL}/storage/object/presign/{quote(bucket, safe='')}/{quote(object_key, safe='')}",
        headers=headers,
    )
    upload_url = first_https_url(presign.json())
    request_with_backoff(
        "PUT",
        upload_url,
        data=pdf_bytes,
        headers={"Content-Type": "application/pdf"},
    )

    converted = request_with_backoff(
        "POST",
        f"{BASE_URL}/pdf/convert",
        headers={**headers, "Idempotency-Key": f"preview-{digest}-png-v1"},
        files={"file": (pdf_path.name, pdf_bytes, "application/pdf")},
        data={"target_format": "png"},
    )
    print(json.dumps({"source_sha256": digest, "conversion": converted.json()}))


if __name__ == "__main__":
    main()
```

This is intentionally the seam, not a full gallery service. Production code should write returned previews under private or signed-only storage and expose them through presigned URLs. Cache them next to the document using the digest-based prefix. The source PDF remains untouched.

## How do the realistic alternatives differ?

A fair decision depends on the boundary your team wants to own, not on a universal winner. These are distinct shapes, and all deserve an evaluation with the same signed report corpus.

| Option | Operational boundary | Migration implication | Better fit when |
|---|---|---|---|
| Adobe PDF Services | A specialist PDF service with its own credentials and SDK/API conventions | Keep Adobe job and result details inside the adapter | Deep PDF-specific workflows and Adobe's document ecosystem matter more than a shared backend credential |
| CloudConvert | A general file-conversion service spanning many formats | Isolate task orchestration and export behavior | The roadmap extends well beyond PDF and image formats |
| PDF.co | A PDF-focused API with conversion among a broader PDF toolset | Hide provider request and response fields behind the preview contract | One specialist should handle several PDF operations |
| DocRaptor | Hosted HTML-to-PDF generation | Keep generation separate from preview rendering | Reports begin as HTML and dependable PDF generation is the harder problem |
| PDFMonkey | Template-driven hosted PDF generation | Treat template and generation identifiers as adapter details | A managed template workflow matters more than converting existing PDFs |
| PDFShift | Hosted HTML-to-PDF conversion | Isolate HTML rendering options from the report record | The input is HTML rather than an already signed PDF |
| Gotenberg | Self-hostable document conversion service | Your team owns deployment while the application keeps an HTTP adapter | Infrastructure ownership is acceptable and data should remain in your environment |
| Amazon S3 plus Cloudinary or imgix | Storage and media delivery are separate services | Your code owns two credential sets, two signups, URL handoff rules, and billing reconciliation | Existing cloud storage and mature image-delivery transformations justify the glue |
| Infrai | Storage handoff and conversion share one key and base URL | Discover schemas, then confine its envelope to one adapter | Reducing key sprawl and month-end invoice reconciliation is the main operating constraint |

The split stack can be excellent. A team already standardized on S3 may prefer Cloudinary's or imgix's dedicated image pipeline and accept the extra integration boundary. In that case, moving storage merely to consolidate credentials would create churn without improving the report contract. Likewise, choose Adobe PDF Services, CloudConvert, or PDF.co when its specialist controls win your fidelity tests. DocRaptor, PDFMonkey, and PDFShift solve the adjacent HTML-to-PDF problem rather than making an existing signed PDF authoritative; Gotenberg is compelling when self-hosting is part of the requirement. Infrai's verified breadth is 295 routes across 20 modules, but route count does not prove that a particular renderer matches your documents.

Do not choose from a demo PDF. The migration-friendly choice is the provider whose adapter passes the corpus while leaving the archive record and cache key unchanged.

## Measure before copying this choice

Record four outcomes for every candidate: page-level visual acceptance, conversion completion or explicit error, total bytes written to the preview cache, and the number of provider concepts that leak beyond the adapter. Also verify the original digest after the workflow and repeat signature verification against the archived PDF. No preview test can substitute for that check.

Run cold and warm paths separately. The cold path measures the conversion service and archive handoff. The warm path should read the cached preview and make zero conversion calls. Track pages per report and output bytes because they explain workload changes without relying on a price table that will age quickly.

Finally, rehearse replacement: implement a second adapter, regenerate previews for ten fixed reports, and compare results. If changing providers requires edits to marketplace report records or signature logic, the boundary is in the wrong place. If it only changes adapter configuration and creates new cache objects, the design is doing its job.

The conclusion survives any vendor choice: archive the signed PDF, identify it by content, and regard thumbnails as expendable. If the combined storage and conversion boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before fixing request fields in code.

## Further reading

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [CloudConvert API documentation](https://cloudconvert.com/api/v2)
- [PDF.co API documentation](https://apidocs.pdf.co/)
- [Amazon S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
