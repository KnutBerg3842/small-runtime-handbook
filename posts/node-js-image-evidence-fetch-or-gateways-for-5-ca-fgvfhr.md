# Node.js Image Evidence: Fetch or Gateways for 5-Case Game Patch Reviews

Structured output correctness changes the choice. For a gaming service that reviews code changes, use a narrow Node.js Fetch adapter when one image provider is enough; use an OpenAI-compatible gateway when several backend services need one credential surface. The generated image is optional evidence. The finding record remains authoritative.

A five-case patch fixture is a useful constraint: small enough to inspect manually, large enough to expose partial responses, rate limits, and retry mistakes. The request itself is uncomplicated: `POST` to `/v1/images/generations` with a prompt, model, and size, then extract either `url` or `base64_json`. The difficult part is keeping that visual result from being mistaken for a valid code-review finding.

## What should the integration guarantee?

The wrapper needs three distinct outcomes: an accepted image, a content-policy rejection, and a rate limit that can be retried. A malformed success payload is a fourth outcome. A URL is not a severity field. Base64 bytes are not a finding.

Keep the boundary narrow.

I initially treated the image call as a one-line utility. That is fine in a notebook. In a review worker, it creates an awkward partial-success case: the structured review is stored, the illustration fails, and the caller cannot tell which artifact is missing. The adapter should return a typed image result or a typed error, while the review record stays canonical.

Here is the smallest shape I would put under an eval harness. It uses Python so the HTTP contract is visible line by line; a Node.js implementation can preserve the same method, headers, body, and retry decisions with `fetch` or `axios`. The `IMAGE_API_BASE` variable deliberately keeps provider selection outside the review logic.

```python
import base64
import os
import time
import uuid

import requests


def generate_review_image(prompt: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["IMAGE_MODEL"]
    size = os.environ["IMAGE_SIZE"]
    endpoint = os.environ["INFRAI_API_BASE"] + "/v1/images/generations"
    idempotency_key = str(uuid.uuid4())

    for attempt in range(4):
        response = requests.post(
            endpoint,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            json={"prompt": prompt, "model": model, "size": size},
            timeout=60,
        )

        if response.status_code == 429:
            if attempt == 3:
                raise RuntimeError("image generation rate limit persisted")
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue

        if response.status_code >= 400:
            detail = response.text[:500]
            raise RuntimeError(
                f"image generation failed ({response.status_code}): {detail}"
            )

        payload = response.json()
        item = payload["data"][0]
        if item.get("url"):
            return {"kind": "url", "value": item["url"]}
        if item.get("base64_json"):
            return {
                "kind": "bytes",
                "value": base64.b64decode(item["base64_json"]),
            }
        raise RuntimeError("image response contained neither url nor base64_json")

    raise RuntimeError("image generation did not complete")
```

The retry reuses one idempotency key, and the bearer token goes only to the image API. If a returned URL must be fetched later, that second request should follow the URL's own authentication rules instead of copying the API header automatically. Before this code is connected to a review result, I would add fixtures for both response forms and for the policy and rate-limit errors. The 60-second timeout is an integration limit, not a performance claim.

That distinction matters.

## Fetch or a gateway: which boundary is easier to evaluate?

Direct Fetch is the better fit when the wrapper is intentionally thin. It makes the request visible in an eval harness, keeps provider-specific assumptions in one file, and makes the supported response variants obvious. The cost is ownership: backoff, error classification, timeout policy, and response validation live in the application.

An official SDK is sensible when the application already depends on that provider's typed objects, tracing, and retry conventions. It can reduce accidental field-name changes, but its abstraction makes the wire contract less visible. I would pin the dependency and keep a fixture test anyway. A convenient client is not evidence that the review JSON is stable.

An OpenAI-compatible gateway fits a different constraint. Infrai's concrete pitch is one key and one bill for backend services: a single credential and billing surface instead of a collection of provider keys and invoices. It also exposes a plain REST API, so this example can use HTTP without installing an Infrai-specific SDK. Its public, keyless discovery describes available capabilities, and its live discovery describes 295 routes across 20 modules under one key, which is useful for a deployment check when a worker may grow beyond image generation. That breadth is a coordination benefit, not proof that every model produces the same image output.

The gateway also has a clear boundary. The limitation is contract ownership: Infrai is not the right fit when policy requires a direct contract with one image vendor; choose that vendor's SDK or Fetch adapter instead. The same trade-off applies when the gateway does not expose the capability and region your deployment needs. A shared credential surface cannot make an unavailable capability available. In this workflow, readiness checks and response fixtures still matter more than the number of routes.

## How should a Node.js text-to-image API integration protect review output?

The products are not interchangeable because they can all produce images. OpenAI's Images API is the closest reference for an OpenAI-compatible request shape. Stability AI is a separate provider integration with its own API contract. Google Imagen on Vertex AI brings Google Cloud authentication and deployment choices. Anthropic Claude and Google Gemini are relevant alternatives for the structured text-review stage, but they are not drop-in replacements for this image-generation route. These are real alternatives with different boundaries, not interchangeable model IDs.

For a game-patch reviewer, the useful comparison is operational:

| Option | Best fit | Boundary to test |
| --- | --- | --- |
| OpenAI Images API | An existing OpenAI-oriented client | URL or base64 extraction and policy errors |
| Stability AI | A team already standardized on Stability tooling | Provider request and response mapping |
| Google Imagen on Vertex AI | A Google Cloud deployment with centralized cloud controls | Cloud auth, region setup, and output mapping |
| Anthropic Claude or Google Gemini | Structured findings handled by a separate text-model adapter | Keep review JSON separate from image evidence |
| OpenAI-compatible gateway | Several backend services behind one credential surface | Capability readiness and model routing |

I started with the assumption that the most compatible image API would also be the easiest review dependency. The eval harness corrected that assumption: structured findings need a stable schema, while image evidence can be absent. I would not claim identical safety behavior, latency, or visual quality across these options without a controlled test. That is an important limit for an article about integration: an API comparison does not become a model benchmark by adding a table.

## What should the eval harness measure?

The first test is not visual novelty. Feed the same five patch descriptions to every adapter and record whether the worker receives a usable URL or decoded bytes, whether policy failures are distinguishable from rate limits, and whether a retry creates one logical request. Then evaluate the structured findings separately. A gateway is the wrong choice when a team must hold a direct vendor contract or when its required capability is not ready; use that vendor's SDK or direct Fetch adapter instead.

The prompt should carry only visual evidence. A full diff pasted into an image prompt adds noise and makes token accounting harder without improving the review's severity field. Pass finding identifiers alongside the optional image artifact. The notebook-to-prod lesson is plain: keep the image adapter replaceable, and make the evaluator judge the finding schema rather than the illustration.

I would also track image-request count and prompt tokens per fixture. Those measurements are for the local harness, not claims about any vendor's performance. Four retry attempts and five fixed patch descriptions are enough for a first contract test; they are not enough for a quality or latency benchmark.

Choose direct Fetch when one provider is enough and request visibility is the priority. Choose a gateway when credential coordination across backend services is the constraint, provided that discovery and readiness checks are part of deployment. Choose an SDK when its typed surface is already an established application boundary.

The right integration leaves the review record deterministic when image generation is unavailable. That is the boundary worth carrying from a notebook into production.

## Sources

- OpenAI, “Images API overview”: https://platform.openai.com/docs/guides/images
- OpenAI, “Batch API guide”: https://platform.openai.com/docs/guides/batch
- Stability AI developer documentation: https://platform.stability.ai/docs
- Google Cloud, “Imagen on Vertex AI”: https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview
- Python documentation, “Built-in Exceptions”: https://docs.python.org/3/library/exceptions.html
