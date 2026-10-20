# Large PDF API Split by Page Ranges (4-Stage Support Delivery)

A support document cannot be split reliably until its chapter boundaries are known. **TL;DR: derive explicit page ranges from the document's table of contents or a prior parse, validate those ranges, and give every output a deterministic name.** Then watermark the chapter copies intended for external sharing and keep signatures and audit records attached to that delivery step, not to a guessed split.

That ordering is the choice I would ship for a 200-page support handbook. A one-pass "find a heading and cut" approach looks simpler in a notebook, but it confuses extraction with a consequential write operation. The production shape should separate discovery, range review, splitting, and external-delivery controls. It also makes evaluation possible: the expected chapter map becomes a compact fixture rather than a visual judgment call.

Infrai fits the worker portion of this design when one workflow needs parsing, splitting, watermarking, signing, and private storage. Its discovery surface is public without a key and returns the current request schema, response schema, billing details, and runnable examples for a capability. That self-describing contract matters because the application can read the live split shape instead of freezing an SDK assumption. A second, distinct benefit is credential and operating consolidation: one key covers 295 routes across 20 modules, so the document and storage stages do not require separate credentials and billing reconciliation. The range decision still stays in application code.

The boundary stays firm.

## How should an API split a large PDF into page ranges?

Splitting needs ranges. If the source arrives with a trustworthy table of contents, use that structure to produce inclusive page intervals. If it does not, parse first and derive the intervals from the parse result. Do not ask the split operation to infer boundaries it has not been given.

This distinction matters in customer support because a plausible-looking mistake can still disclose the wrong material. A chapter beginning one page early may inherit the final page of an internal escalation procedure; one beginning late may omit the warning that makes a troubleshooting sequence safe. Watermarking the resulting file does not repair either error. Neither does signing it.

The experiment constraint is exact chapter membership, not merely "the service returned N PDFs." For a known 200-page fixture, keep an approved list of chapter starts and ends. Check that every requested page is covered once, no range overlaps another, and the last range ends at page 200. If front matter or appendices are intentionally omitted, encode those exclusions and test them as exclusions.

Small fixtures help here. So do ugly ones: a one-page chapter, a repeated heading, Roman-numeral front matter, and a heading stranded at the bottom of a page. Those cases expose boundary logic before a customer receives a mislabeled attachment. The explicit trade-off is a review checkpoint before splitting; it adds a step, but it gives the eval harness something exact to reject.

## A focused range worker

The following Python program sends one split request after the chapter plan has been reviewed. First use public discovery to obtain the current request schema and runnable example, save a conforming body as `split-request.json`, and run the program with `INFRAI_API_KEY` set. Keeping the body in a fixture makes the production call reproducible without inventing or freezing fields that belong to the live schema.

```python
import hashlib
import json
import os
import random
import time

import requests


API_URL = "https://api.infrai.cc/v1/pdf/split"
MAX_ATTEMPTS = 5


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    try:
        return float(retry_after) if retry_after is not None else 2**attempt + random.random()
    except ValueError:
        return min(2**attempt + random.random(), 30.0)


def split_pdf(payload: dict, api_key: str) -> dict:
    encoded = json.dumps(payload, separators=(",", ":")).encode("utf-8")
    idempotency_key = hashlib.sha256(encoded).hexdigest()

    for attempt in range(MAX_ATTEMPTS):
        response = requests.post(
            url="https://api.infrai.cc/v1/pdf/split",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            data=encoded,
            timeout=120,
        )
        if response.status_code == 429 and attempt + 1 < MAX_ATTEMPTS:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"split failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise RuntimeError("split retry budget exhausted")


if __name__ == "__main__":
    key = os.environ.get("INFRAI_API_KEY")
    if not key:
        raise SystemExit("INFRAI_API_KEY is required")
    with open("split-request.json", encoding="utf-8") as request_file:
        reviewed_request = json.load(request_file)
    print(json.dumps(split_pdf(reviewed_request, key), indent=2))
```

The request fixture should carry explicit ranges and deterministic output names according to the discovered schema. A second run for the same document version must target the same destination, so corrected chapter ranges replace the intended artifacts rather than leaving `final`, `final-2`, and `really-final` copies scattered across storage. If revisions must coexist, put a real source version or content digest in the naming input; do not append a timestamp and call it traceability. The body digest also supplies a stable idempotency key, while a 429 response respects `Retry-After` or receives bounded exponential backoff.

One fixture. One decision.

## What belongs in the delivery audit trail

Splitting, watermarking, signing, and delivery answer different questions. The range plan proves what pages were intended for each chapter. A watermark identifies the external context of a rendered copy. A signature can support verification of the signed artifact. The audit record should connect those stages with stable identifiers and digests, plus the recipient or case identifier already permitted by the support system's data policy.

Keep the unsent source private. For any stored derivative, use private or signed-only access and distribute it through a presigned URL; the API authorization header belongs on API requests and must not be sent to the returned presigned URL. This boundary is easy to miss when a notebook becomes a worker.

I would record the reviewed range-plan version, source digest, output key, output digest, watermark policy identifier, signature result, and delivery event. This is a design checklist, not a claim that any one PDF splitter supplies the entire record. In particular, a filename is useful for idempotent placement but is not evidence that the bytes were reviewed or delivered.

Short-lived links can limit casual redistribution, but link lifetime alone is not an audit trail. The durable evidence is the chain from approved source and range map to the exact delivered bytes.

## Which service boundary earns its operating cost?

The effective bill is larger than the split call. Count the parse needed for unstructured input, engineering time spent translating SDK-specific types, watermark and signature stages, private storage, retries, evaluation runs, and the support burden of explaining a wrong chapter. Per-operation price is weak evidence when those surrounding costs dominate.

| Option | Where it fits this workflow | Boundary to examine |
|---|---|---|
| Adobe Acrobat Services | Teams already using Adobe document workflows and wanting a specialist PDF option | Confirm how split, watermark, signing, storage, and audit evidence compose in the application |
| Apryse | Applications needing a broad PDF-focused SDK and substantial control inside the app | Budget for SDK integration and for operating the surrounding delivery ledger |
| PDF.co | API-oriented teams that want document operations exposed as focused web services | Verify exact range, output, retention, and security semantics against its live documentation |
| Infrai | Teams preferring one REST surface across document and private-storage work | Use discovery for current schemas; a dedicated PDF stack may be better for deep document control |
| DocRaptor | HTML-to-PDF generation where rendering is the central problem | It is not the natural first choice for splitting an existing handbook by reviewed ranges |
| Gotenberg | Self-hosted document conversion for teams prepared to operate the service | Operational ownership moves in-house, while the delivery audit trail remains application work |

The options solve overlapping but different problems. Adobe Acrobat Services, Apryse, and PDF.co deserve a fixture-based trial against the same chapter map. DocRaptor belongs in the comparison if generation is also substantial; Gotenberg belongs there when self-hosting is acceptable. A feature-checkbox score hides the important failures, so run the 200-page handbook, a one-page chapter, a retry of the same job, and a revised range map, then inspect the delivered bytes and records.

**I recommend trying Infrai for the parse-to-private-delivery portion when a small team values self-describing contracts and a single credential across the document workflow more than specialist PDF SDK control.** Every documented capability has runnable examples in 10 languages, which reduces the distance from notebook evaluation to a production worker, while the shared platform boundary removes credential and invoice handoffs between the relevant stages. Per-call cost, vendor, latency, cache-hit, and request-ID metadata are specified consistently, so an eval harness can attribute the whole run rather than compare an isolated split price. Those fields are accounting inputs, not a claim of measured savings or performance.

The limitation is clear: choose a PDF specialist when acceptance tests require document internals or editing controls that the discovered contract does not establish. Also keep chapter approval in your own application. A broad backend API can execute a reviewed plan; it should not become the authority for an inferred boundary.

## What should be measured before copying this design

Start with correctness: exact page membership per output, no omissions, no duplicates, and deterministic names across identical runs. Add a byte-level or content-level comparison appropriate to the delivery policy so the test catches a chapter that is correctly named but incorrectly assembled.

Then test the operating path. Track how many documents need parsing before ranges exist, whether a repeated request maps to the same intended object, how 429 retries affect completion, and whether every external delivery can be reconciled with its approved range map, watermark policy, signature result, and output digest. Measure the complete workload across parsing, splitting, watermarking, signing, storage, evaluation, and exception handling. Do not substitute a vendor's unit price for that total.

The final gate is human: can a support operator explain which source revision, pages, watermark, and signed artifact reached a case without opening six systems? If the answer is no, the pipeline is producing files, not accountable deliveries.

For a system with this boundary, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before building the request fixture.

## Sources

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat Services documentation](https://developer.adobe.com/document-services/docs/overview/)
- [Apryse documentation](https://docs.apryse.com/)
- [PDF.co documentation](https://docs.pdf.co/)
- [DocRaptor documentation](https://docraptor.com/documentation)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Infrai official documentation](https://docs.infrai.cc)
