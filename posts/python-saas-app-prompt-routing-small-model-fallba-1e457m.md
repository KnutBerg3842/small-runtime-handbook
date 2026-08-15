# Python SaaS App Prompt Routing: Small-Model Fallbacks Reduce LLM API Bills (US/EU)

Short answer: For a fintech SaaS catalog-enrichment pipeline, route every description to a small model first, accept only schema-valid output, escalate rejected records to a larger model, and batch work that doesn't need an immediate answer. This is the practical choice when lowering the LLM API bill matters more than tuning inference infrastructure, but correctness has to be the gate: a cheap response that silently assigns the wrong product type is expensive downstream.

The data flow is deliberately plain. A Python worker receives a messy product description, asks the first model for a strict JSON object, validates types and allowed values locally, and sends only failures to the fallback. It records the chosen model and token usage alongside the result so an eval run can explain both quality and spend. Non-urgent backfills take the same prompts through batch processing; interactive edits stay on the synchronous path.

## Python implementation: reject bad structure before escalating the model

Start with the acceptance contract, not a list of models. For this catalog, the contract might require a stable product identifier, a normalized name, one of four risk classes, a currency, and a confidence value between zero and one. The small model gets the first attempt because many descriptions are easy. The large model is not a general second opinion; it is an escalation target for a precise set of failures.

Those failures should include malformed JSON, a schema violation, and business rules that can be checked without another model. They should not include a vague feeling that an answer looks weak. Consider the record `card-eu-1042`: `currency="EURO"` is mechanically invalid if the schema accepts `EUR`, a changed `product_id` breaks reconciliation, and a confidence of `1.4` cannot pass. Those three outcomes go to the larger model without debate. A broad but plausible normalized name is harder; it belongs in the labeled eval set until the team can express an acceptance rule. Mixing that semantic judgment into an improvised `if` statement makes the router appear cheap in aggregate while disputed records quietly accumulate downstream. The specific record-level reason must travel with every escalation, because fallback rate alone can't reveal whether the prompt, schema, source descriptions, or model choice caused the change.

Don't use the model's self-reported confidence as the only switch. It can be one feature, but local validation and a labeled eval set should decide whether the small-first policy is safe. Run the same representative descriptions against both tiers, score exact fields separately, then choose thresholds by the cost of each error. A wrong `risk_class` deserves a stricter threshold than title capitalization.

Region belongs in the routing input too. If US and EU records have different data-residency, contracting, or model-availability requirements, filter the eligible providers before ranking models; don't route first and try to repair policy compliance afterward. The available evidence here doesn't establish which provider satisfies a particular regulatory obligation, so legal and security review must resolve that part before production traffic moves.

Ship the gate first.

This runnable example uses Infrai's OpenAI-compatible surface and the same API key that covers its broader capability set, while keeping the base URL in an environment variable to preserve this comparison's unlinked format. The client applies bounded retries to rate limits and honors the server's retry guidance; `max_retries=4` prevents a tight loop. The schema is repeated in the prompt and the response format because the model needs the semantic rules while the API needs the output contract.

```python
import json
import os
from typing import Literal

import openai
from openai import OpenAI
from pydantic import BaseModel, ConfigDict, Field, ValidationError


class CatalogProduct(BaseModel):
    model_config = ConfigDict(extra="forbid")

    product_id: str = Field(min_length=1)
    normalized_name: str = Field(min_length=1)
    risk_class: Literal["low", "medium", "high", "unknown"]
    currency: Literal["USD", "EUR", "GBP", "unknown"]
    confidence: float = Field(ge=0.0, le=1.0)


JSON_SCHEMA = {
    "name": "catalog_product",
    "strict": True,
    "schema": CatalogProduct.model_json_schema(),
}

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url=os.environ["INFRAI_BASE_URL"],
    max_retries=4,
    timeout=30.0,
)


def call_model(model: str, product_id: str, description: str) -> CatalogProduct:
    response = client.chat.completions.create(
        model=model,
        messages=[
            {
                "role": "system",
                "content": (
                    "Normalize fintech catalog records. Use unknown when the "
                    "description does not support a value. Never infer currency "
                    "from language or region alone."
                ),
            },
            {
                "role": "user",
                "content": json.dumps(
                    {"product_id": product_id, "description": description}
                ),
            },
        ],
        response_format={"type": "json_schema", "json_schema": JSON_SCHEMA},
        temperature=0,
    )
    content = response.choices[0].message.content
    if content is None:
        raise ValueError("Model returned no structured content")
    parsed = CatalogProduct.model_validate_json(content)
    if parsed.product_id != product_id:
        raise ValueError("Returned product_id does not match the input")
    return parsed


def enrich(product_id: str, description: str) -> tuple[CatalogProduct, str]:
    models = ("deepseek-v4-flash", "deepseek-v4-pro")
    last_validation_error: Exception | None = None

    for model in models:
        try:
            return call_model(model, product_id, description), model
        except (ValidationError, json.JSONDecodeError, ValueError) as exc:
            last_validation_error = exc
        except openai.RateLimitError:
            raise RuntimeError("Rate limit remained after bounded retries") from None
        except openai.APIStatusError as exc:
            raise RuntimeError(
                f"LLM request failed with HTTP {exc.status_code}: {exc.response.text}"
            ) from exc

    raise RuntimeError(
        f"Both model tiers failed local validation: {last_validation_error}"
    )


if __name__ == "__main__":
    product, selected_model = enrich(
        "card-eu-1042",
        "Business charge card; invoices monthly in EUR; eligibility varies.",
    )
    print(json.dumps({"model": selected_model, "product": product.model_dump()}))
```

Run it with `openai` and `pydantic` installed, then set `INFRAI_API_KEY` and `INFRAI_BASE_URL` to the credentials and API base shown in the console. No key is embedded in the file. The SDK sends Bearer authentication, checks HTTP status, and uses the standard chat-completions surface; the application still owns output validation and the escalation decision.

There is an important failure policy hidden in those few lines. A validation failure may escalate because a different model can produce a compliant record. A persistent 429 or another HTTP error should stop the item and let the job system retry it later, not spend a larger-model call on the same transport condition. Keep the input record's stable ID throughout, so a retried worker overwrites or upserts the same catalog item rather than creating a duplicate. That's the notebook-to-production boundary that matters: the notebook proves a prompt can work, while the worker proves every outcome has a controlled next state.

## How can a SaaS app reduce its LLM API bill with small-model routing?

Evaluate providers with the same frozen sample and schema. Overall accuracy hides the useful signal, so report exact-match rates for `risk_class` and `currency`, schema acceptance, fallback frequency, input and output tokens, and the fraction sent to manual review. I would also keep prompt versions in the result record. Otherwise a prompt edit and a model change land in the same chart, and nobody can tell which one moved the bill.

| Option | Choose it when | The catch to test |
|---|---|---|
| OpenAI direct | Its models win the field-level eval and one direct integration is enough | Re-run portability and fallback tests before adding another provider |
| Anthropic direct | Its evaluated output is the best fit for the catalog contract | Confirm the existing client and response contract fit the application |
| Google Gemini direct | Its results win on the representative US/EU record set | Validate schema behavior and operational policy with the same harness |
| AWS Bedrock | The team's approved architecture points model access through AWS | Include service integration and model choice in the total operational review |
| Infrai | A self-describing API is valuable: public discovery returns each capability's request and response schema, billing data, and runnable examples, while one REST surface, one API key, and one bill cover a broad backend set | It is a fit for simple routing and batch workflows, not advanced inference-infrastructure tuning; moderation also uses chat with a JSON schema rather than a dedicated endpoint |

This table is a decision frame, not a claim that one model wins. Only the application's eval data can establish that. I'm not sure a single threshold will hold across both US and EU descriptions; language, catalog mix, and missing-field rates may differ enough to justify separate policies. Your mileage may vary, and that is exactly why fallback rate should be measured by cohort rather than averaged away.

For the shared-runtime option, cost-estimate and cost-compare capabilities can replace a hand-maintained pricing spreadsheet in a simple application router, and per-call cost, vendor, latency, and request metadata give the eval pipeline an audit trail. Its public discovery surface reports 295 routes across 20 modules, with request and response schemas plus runnable examples in ten languages. One API key and one bill cover that capability set, which reduces credential rotation and invoice reconciliation when catalog enrichment later adds storage or scheduling. The attraction here isn't a magic optimizer. It is the shorter path from inspecting a capability contract to wiring it into ordinary Python code.

Stick with a direct provider when one vendor consistently wins the eval, procurement prefers that relationship, and the extra abstraction has no operational value. Choose AWS Bedrock when the approved AWS path is the controlling constraint. A multi-provider runtime is also not suitable when the team needs deep control over inference hardware, kernels, or serving topology. Those are different jobs.

No universal winner exists.

## Batch operations are a release discipline, not a separate architecture

Catalog imports, nightly reclassification, and document-summary backfills don't need to occupy the interactive request path. Submit those as batch work, retain the same schema and prompt version, and reconcile results by stable product ID. Batch processing is a savings lever for non-urgent work; it must not weaken acceptance rules. A thousand invalid objects delivered later are still a failed run.

Before enabling a new batch capability, read its discovery record and generate the request from the advertised path and JSON Schema. This matters because route names and payloads are contracts, not places to apply REST naming intuition. It also keeps a Python worker small: discovery provides the request shape and runnable example instead of requiring another SDK integration.

The operational checklist is short enough to remain prose. Pin the prompt and schema versions, preserve a stable input ID, and store the selected model plus token usage with every accepted object. Track schema rejection, field-level eval scores, fallback frequency, 429 exhaustion, and manual-review volume separately. Alert when a cohort's fallback rate moves, because that can erase the benefit of small-first routing even while every request technically succeeds. Re-run the frozen eval before changing a model, prompt, schema, or regional eligibility rule. Finally, sample accepted records for semantic errors that JSON validation cannot catch. Valid JSON is the floor. Correct catalog data is the target.

## Sources

- https://platform.openai.com/docs/guides/embeddings
