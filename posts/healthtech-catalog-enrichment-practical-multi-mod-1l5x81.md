# Healthtech Catalog Enrichment: Practical Multi-Model API Choices for Small Teams

Short answer: for a small team enriching a healthtech catalog, choose the narrowest multi-model API contract that preserves provider portability, then prove it with a fixed evaluation set before putting it in production. The winning design is not the API with the longest feature list. It is the one that lets your catalog pipeline change providers without rewriting normalization, validation, audit, and retry logic.

Catalog enrichment looks harmless until the input arrives. A product description may contain a brand name, a dosage, a unit, a missing manufacturer, and a typo in the same sentence. The output still has to fit a stable schema, preserve uncertainty, and avoid inventing a clinical attribute. A fluent answer is not enough.

This is an experiment note, not a leaderboard. The simple approach is one provider SDK called directly from a notebook, with its response dictionary passed into the database. The practical approach is a small Python boundary, a provider-neutral request shape, explicit validation, and an evaluation harness that compares behavior rather than prose. It takes a little more setup. It buys an exit path.

## Why does catalog enrichment expose provider lock-in so quickly?

The first lock-in point is usually the response shape. A notebook starts with `response.choices[0].message.content`, and six weeks later that expression is scattered through batch jobs, review tools, and tests. The second is prompt behavior: one provider's role handling, tool syntax, or structured-output convention quietly becomes part of the application contract. The third is operational. Credentials, rate-limit handling, usage accounting, tracing, and retry policy end up coupled to the same client library.

Healthtech adds a sharper boundary. Under the HIPAA Security and Privacy Rules, a team has to reason about safeguards and permitted handling of protected health information; a model gateway does not make that assessment for the team. Catalog text may be public, but an enrichment pipeline can still receive supplier notes, internal identifiers, or a description that reveals more than its author intended. Redact before inference where possible, log the minimum useful payload, and make retention an explicit decision.

The failure mode is subtle: the output passes a visual review while the provenance disappears. If the model turns “latex-free gloves, medium, box of 100” into a product record, the system should retain the source text, extracted fields, confidence or review state, and the model policy used. When a field is wrong, an operator needs to tell whether retrieval, prompting, parsing, or the source record caused it. That means the worker needs an evidence trail even for fields that look obvious: the exact excerpt supporting `unit_count`, the source record version, the prompt policy, the schema version, the response validation result, and the reviewer decision should travel together. Without that trail, a later provider swap cannot be evaluated cleanly because a score change may be caused by changed source data or changed review rules rather than model behavior. This is where a small team can lose more time than it expected: the model call is easy to replace, but the unrecorded assumptions around it are not.

Keep the contract small.

For this job, the common contract can be a text input, a requested schema version, a model policy, and a structured result. Provider-native extras belong behind optional capabilities. If the shared contract starts exposing every provider's special case, portability becomes a label on a very large adapter.

## What failure modes should a small team test before choosing a multi-model API?

Start with representative catalog records, not a model comparison page. Build a small corpus containing ordinary descriptions, missing units, contradictory quantities, multilingual fragments, malformed markup, and records that should be sent to human review. The corpus does not need to be huge. It needs to contain the awkward cases that make a direct integration look successful in a notebook and unreliable in a batch job.

Define the result before selecting the provider. A useful record might contain `name`, `manufacturer`, `unit_count`, `material`, `is_sterile`, `source_excerpt`, and `review_reason`. Every field needs a rule for unknown values. “Not stated” must not become a confident `false`, and a missing dosage must not be filled from general medical knowledge. Schema validity is a gate, not a quality score.

Then run the same cases through each candidate behind one application interface. The point is not to make OpenAI, Claude, and Gemini produce identical wording. The point is to compare field accuracy, abstention behavior, JSON validity, latency, token use, and review volume while keeping business logic fixed. `tiktoken` can estimate tokens for tokenizers it supports, but the runtime's usage record should be the accounting source for an actual request.

I keep one `429` fixture in the harness because rate-limit behavior is part of portability. A retry that changes a request body, duplicates a write, or hides the original trace is a pipeline defect even when the final answer looks fine. I'm not sure which provider-specific feature the next release will need; that uncertainty belongs in the design as an explicit escape hatch.

## What should the Python boundary own before a model call?

The adapter should own translation, timeout policy, response extraction, and usage metadata. The catalog service should own schema rules, provenance, review routing, and persistence. That split keeps a provider change from touching the database code. It also makes notebook-to-production progress visible: the notebook calls the same boundary, while the production worker adds queueing, idempotency, and observability around it.

Here is a deliberately plain boundary. It does not pretend that different models have identical semantics, and it does not hide validation behind a vendor SDK. The provider-specific transport can be implemented separately; this example shows the contract the rest of the application consumes.

```python
from dataclasses import dataclass
from typing import Any, Protocol


@dataclass(frozen=True)
class EnrichmentRequest:
    text: str
    schema_version: str
    policy: str


@dataclass(frozen=True)
class EnrichmentResult:
    fields: dict[str, Any]
    source_excerpt: str
    review_reason: str | None
    provider_label: str
    usage: dict[str, int]


class ModelAdapter(Protocol):
    def enrich(self, request: EnrichmentRequest) -> EnrichmentResult:
        """Return a validated, provenance-preserving enrichment result."""


def enrich_catalog_row(
    adapter: ModelAdapter, description: str, policy: str
) -> EnrichmentResult:
    request = EnrichmentRequest(
        text=description,
        schema_version="catalog-item-v1",
        policy=policy,
    )
    result = adapter.enrich(request)

    if not result.source_excerpt:
        raise ValueError("missing source excerpt")
    if result.review_reason is None and not result.fields:
        raise ValueError("empty enrichment without a review reason")
    if any(value == "unknown" for value in result.fields.values()):
        raise ValueError("use null or a typed uncertainty state, not a string sentinel")
    return result
```

The important lines are the boring ones. `schema_version` gives an eval result a durable meaning. `source_excerpt` makes a field auditable. `review_reason` gives the model a safe way to abstain. `usage` keeps prompt-cost awareness in the same record as task quality. The adapter may translate a common request into different provider calls, but the application never needs to know which role syntax or response envelope was used.

## Which trade-offs matter more than a feature checklist?

Use a scorecard that reflects the catalog job. A provider-neutral layer is useful only when its operational simplification outweighs the expressive features you would lose at the boundary.

| Decision area | Portable design | Direct-provider design | Test before choosing |
|---|---|---|---|
| Request shape | One small schema and policy object | Native controls exposed to application code | Count business-logic edits in a provider swap |
| Structured output | Validate locally and route failures to review | Rely more heavily on native structured-output behavior | Measure valid schema rate and missing-field rate |
| Prompt changes | Version prompts and policies centrally | Tune each provider's prompt conventions | Re-run the same corpus after each change |
| Operations | Shared tracing, retries, budgets, and redaction | Provider-specific telemetry and controls | Inspect 429, timeout, malformed-response, and retry fixtures |
| Specialized capability | Add an explicit optional adapter | Use the provider-native path | Confirm the capability is essential, then isolate it |

The catch is that portability is not free. A common API can flatten useful differences, and a shared policy may be less expressive than a native feature. It is not suitable when the product depends on a provider-specific control, a modality outside the chosen runtime's supported contract, or a regional and contractual requirement that has not been verified independently. In those cases, use a direct adapter for that narrow path and keep the portable interface for ordinary enrichment.

Do not choose on price alone. A consolidated credential and usage view can simplify administration, but neither removes token costs nor predicts catalog accuracy. The selection should survive a change in model, a change in schema, and a change in the team member who owns the worker.

## What evaluation loop turns a notebook into a dependable catalog worker?

The evaluation set should score facts, uncertainty, and operations separately. For each record, check exact or normalized field matches, whether unsupported attributes remain unknown, whether the source excerpt supports the extracted value, and whether the item was routed for review when the evidence was weak. A human reviewer can label a small difficult slice; that slice is often more informative than a large collection of easy product descriptions.

Add failure fixtures. Include an empty description, a description with two conflicting quantities, invalid JSON, a timeout, a 429, and a response containing an unexpected field type. The expected behavior is part of the test: retry only safe work, preserve the original request identity, reject malformed output, and send ambiguous medical attributes to review. Never convert an exception into a plausible catalog value.

For batch work, idempotency matters as much as model quality. Give each source record a stable enrichment key, store the policy and schema version with the result, and make a retry update the same logical attempt rather than create a duplicate. Trace the request from source record to model call to reviewer decision. Logs should answer what happened without retaining more sensitive text than the investigation requires.

The final experiment is a provider-swap drill. Route one slice through another available model policy, compare the scorecard, and inspect the number of edits outside the adapter. A useful result is not identical prose. It is preserved schema behavior, explainable changes in review volume, and a small diff in application code.

No magic.

The replacement should be boring enough that a reviewer can see what changed.

Measure first.

For a lean Python team, that evidence is the practical selection method. It leaves room for native capabilities when they earn their complexity, while the everyday catalog path stays replaceable. The design wins when a provider change is a controlled experiment instead of a rewrite.

## References

- OpenAI `tiktoken`: https://github.com/openai/tiktoken
- HIPAA Security and Privacy Rules, 45 CFR Part 164: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
