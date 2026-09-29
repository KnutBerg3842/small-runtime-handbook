# SMS Notification Service Ownership: Web App Batch Polling and Suppression

A health notification system has one constraint that changes the SMS service decision: a recipient known to be invalid must stay suppressed across every batch, retry, and template revision. **Short answer:** keep templates, consent state, suppression decisions, and delivery correlation IDs in your web app; treat the notification service as a replaceable delivery adapter. If no webhook is available, require queryable per-message status and use polling with bounded backoff.

That choice gives a healthtech team one place to explain why an appointment reminder was attempted, skipped, or stopped. It also avoids tying clinical workflow copy to a provider dashboard that cannot participate in code review or an evaluation harness.

The tempting first draft sends each rendered string and records only the batch ID. It looks fine in a notebook. It fails the first serious test: one invalid destination can reappear in the next batch because the application has no durable recipient-level decision. Fix the ownership boundary before comparing feature matrices.

## Should an SMS notification service own a web app's batch templates?

Template ownership decides where review, versioning, and reproducibility live. For appointment and care-team alerts, the application already knows the workflow event, locale, communication permission, and audience. Rendering there lets a test fixture pin all four inputs to a specific template version. A provider-hosted template can still deliver text, but it moves a critical artifact outside the same deployment and evaluation path as the code that selected it.

I use a narrow rule: if changing a sentence can change user behavior or compliance review, the template belongs beside the application code. The delivery adapter receives rendered content plus an idempotency key; it does not decide what the patient should read. This costs engineering time because localization and rendering need an owner. The payoff is a clean test surface, portable content, and prompt-cost control if an AI system drafts optional copy upstream. Generated text should pass deterministic policy checks before it reaches the send queue.

This boundary also keeps comparison honest. Ask candidates to transport the same fixture rather than comparing their template editors. The experiment then measures the capability you need: acceptance, status retrieval, failure classification, and suppression inputs.

It is a sharp line.

## Build suppression before retry

A suppression record is application state, not a log line. Store a normalized destination key, channel, reason, source message, observed status, and effective timestamp. Keep raw addresses and phone numbers out of routine logs; retain only the identifier required to correlate the event under your organization's data policy.

Then make eligibility a gate in front of queue insertion. A retry worker must call the same gate. So must a later campaign. Otherwise, the system can correctly stop one failed job and still contact the same invalid recipient tomorrow.

The hard trade-off is reversibility. A temporary delivery failure should not silently become a permanent recipient ban, while an explicit opt-out should not expire because a retry timer elapsed. Do not collapse those meanings into a Boolean named `suppressed`. Represent the reason and let policy map reasons to actions.

For a concrete review, trace one appointment reminder all the way through the system. The web app renders version 12 of the template, assigns an internal notification ID, checks the recipient state, and submits the allowed message. The service returns a remote ID. A polling worker later maps its status to the application's vocabulary. If the destination is classified as invalid, the policy layer writes a reasoned suppression record before the next batch is built. Now replay the same workflow with a retry, a process restart, and a changed template. Every outcome should remain explainable from stored application state, with no dashboard-only fact needed to reconstruct it.

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Protocol


class Eligibility(StrEnum):
    ALLOW = "allow"
    INVALID = "invalid"
    OPTED_OUT = "opted_out"


@dataclass(frozen=True)
class DeliveryRequest:
    recipient_key: str
    rendered_body: str
    template_version: str
    idempotency_key: str


class DeliveryAdapter(Protocol):
    def send(self, request: DeliveryRequest) -> str: ...

    def get_status(self, message_id: str) -> str: ...


def submit(
    request: DeliveryRequest,
    eligibility: Eligibility,
    adapter: DeliveryAdapter,
) -> str | None:
    if eligibility is not Eligibility.ALLOW:
        return None
    return adapter.send(request)
```

The code makes a useful test obvious: pass an invalid recipient and assert that `send` was never called. Add fixtures for an opted-out recipient, a duplicated idempotency key, and a template version that fails review. Those tests are cheaper and more stable than inferring policy from delivery logs after a batch runs.

## Polling is a state machine, not a loop

Without webhooks, the application must retain enough information to resume status collection after a process restart. Persist the provider message ID, internal notification ID, attempt count, next-check time, and last observed state. Poll only nonterminal records, apply bounded backoff, and put a time budget around the whole reconciliation job.

Stop cleanly.

The dangerous implementation is `while status != delivered`: it can hammer a status endpoint, run forever on an unknown state, and lose progress when a worker dies. A better worker claims a limited page of due records, queries each once, translates the remote status into a small internal vocabulary, and schedules the next check. Unknown values should remain visible for investigation rather than being guessed into success or permanent failure.

Your internal vocabulary should be smaller than any provider's vocabulary but richer than `success` and `failed`. A practical model distinguishes queued, accepted, delivered, temporary failure, permanent failure, and unknown. The adapter owns translation. Suppression policy consumes only the internal result, so changing services does not rewrite recipient policy.

For batch alerts, completeness matters more than a fast-looking average. Track how many messages remain nonterminal at the reconciliation deadline, how old the oldest unresolved message is, and how many permanent failures produced a suppression decision. Measure duplicate send attempts separately; a clean status table can hide duplicate transport calls.

## Evaluate the boundary with replayable fixtures

Run the same small corpus through every candidate adapter. Twenty to fifty synthetic recipients is enough to expose architectural gaps without pretending to be a throughput benchmark. Include allowed, invalid, opted-out, duplicate, temporarily failed, and never-resolved cases. Use synthetic destinations approved for testing; no patient data belongs in this harness.

| Check | Evidence to retain | Decision signal |
|---|---|---|
| Template control | Rendered body hash and version | Can a deployment reproduce the exact message? |
| Batch submission | Internal and remote IDs | Can every attempt be correlated? |
| Polling recovery | Persisted next-check state | Can a restarted worker continue safely? |
| Invalid recipient | Reasoned suppression record | Will later batches skip the destination? |
| Unknown status | Visible unresolved record | Does the system avoid inventing success? |
| Duplicate defense | Stable idempotency key | Can a retry avoid a second logical send? |

Keep latency and request volume in the report, but do not let them erase correctness. A fast adapter that cannot expose a durable per-message result does not satisfy a polling-only design. Likewise, a polished template editor is not an advantage when reviewed templates must live in the repository.

This is where an eval-driven workflow earns its keep. The corpus becomes a regression suite for adapter upgrades, status-mapping changes, and new channels. It can run in CI with a fake transport, then run against a supported test environment before release. No hand-edited dashboard state is required to explain the result.

The same test shape also fits nonprofit volunteer coordination, where a web app may send batch alerts to US and EU volunteers. The message content and organizational policy differ from healthtech, but the comparison axis does not: template ownership, recipient eligibility, polling recovery, and auditable suppression still belong in the evaluation. Do not infer that one region's consent or messaging rules apply to another; have counsel or the responsible policy owner define those rules, then encode the approved result as application policy.

No guesswork.

## What to measure before copying this design

Start with three ratios: suppressed attempts blocked before transport, messages unresolved at the reconciliation deadline, and duplicate logical notifications. Add counts by suppression reason, because one blended rate cannot tell an opt-out from an invalid destination. For the template boundary, record the percentage of sends whose rendered content can be reproduced from a committed version and fixture inputs.

Watch the operational cost of polling too: status queries per submitted message, queue age, and worker runtime. Those numbers determine whether backoff and page size fit the batch window. They are more actionable than a changing public price because they describe load created by your own design.

The final service choice should follow the experiment: select an adapter only after it can carry application-owned content, return a stable correlation ID, expose per-message status for polling, and provide enough failure information for policy to distinguish retry from suppression. Keep consent and suppression authoritative in the application. That leaves the health workflow explainable even when the transport changes.

## Sources

- https://resend.com/docs/introduction
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
