# Python Mail Probes — Checking DNS Outcomes and Configuration Health

TL;DR: treat a DNS lookup as a preflight check and a controlled mail probe as the decision signal. During a property-management zone migration, a resolver can return the record you expect while a recipient still handles mail differently; a configuration-health monitor needs both observations, tied to the same domain and time window.

For a portfolio of apartment communities, the risky change is rarely the zone-file edit alone. The risk is declaring the move healthy before leasing inquiries and maintenance notices have evidence of the expected authentication policy and delivery path. RFC 7489 defines DMARC policy discovery at `_dmarc.<domain>` and the receiver-side processing that follows. That makes DMARC a useful, observable boundary for this job.

## How should I turn DNS record checking into configuration-health outcomes?

A recursive resolver reports its cached view, not a universal truth. RFC 1034 describes cached data as being removed when its TTL expires, and RFC 2308 extends caching behavior to negative answers. A fresh authoritative answer, a corporate recursive answer, and a recipient-side resolver can therefore disagree for a while after a delegation or TXT change.

That is why I would record resolver evidence with the resolver identity, queried name, response status, answer values, TTL, and timestamp. A monitor that stores only `pass` loses the clue needed to separate a bad zone from a stale cache or a negative cache entry.

Do not turn this into a race to query every public resolver. Pick the recursive paths that matter to the property-management workflow: the organization network, a probe that represents the recipient environment where permission exists, and the authoritative path for diagnosis. Compare like with like. A cached `NXDOMAIN` and a missing expected TXT answer are different failure modes.

## A small Python model for joining records to outcomes

The runnable part of the monitor can stay modest. Feed it normalized DNS observations and controlled-message outcomes collected by the surrounding scheduler or inbox instrumentation. The code below deliberately does not send mail; sending belongs behind an approved delivery boundary, while this step makes the decision rule inspectable and cheap to evaluate in a notebook or a production job.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta


@dataclass(frozen=True)
class DnsObservation:
    domain: str
    resolver: str
    observed_at: datetime
    status: str
    values: tuple[str, ...]


@dataclass(frozen=True)
class ProbeOutcome:
    domain: str
    observed_at: datetime
    delivered: bool
    dmarc_pass: bool | None


def configuration_health(
    dns: list[DnsObservation],
    outcomes: list[ProbeOutcome],
    expected_dmarc_fragment: str,
    now: datetime,
) -> str:
    recent_dns = [item for item in dns if now - item.observed_at <= timedelta(hours=2)]
    recent_outcomes = [item for item in outcomes if now - item.observed_at <= timedelta(hours=2)]
    record_seen = any(
        item.status == "NOERROR"
        and any(expected_dmarc_fragment in value for value in item.values)
        for item in recent_dns
    )
    authenticated_delivery = any(
        item.delivered and item.dmarc_pass is True for item in recent_outcomes
    )

    if record_seen and authenticated_delivery:
        return "healthy"
    if record_seen:
        return "record-present-outcome-pending"
    return "dns-evidence-missing"
```

The two-hour window is an example policy, not a DNS constant. Its job is to stop an old successful message from masking a fresh zone change. The return value is intentionally ternary: a visible record with no successful outcome should trigger investigation, not a false green.

Cache state is part of the result.

This boundary also keeps the monitoring bill predictable. Store small normalized observations and aggregate by domain, resolver class, and time bucket before an evaluation job summarizes them. For an AI-assisted operations workflow, I would use the resulting table as evidence for an alert explanation, then require the raw observations to remain available for review. The model can summarize a mismatch; it should not invent the measurement.

Keep the two evidence streams separate in storage. A DNS observation has its own timestamp and resolver class. A message outcome has a message token and recipient-side result. Joining them only after collection makes stale records visible instead of hiding them behind a boolean. It also keeps a notebook experiment close to production: append observations, rerun the evaluator, and inspect why its state changed. The evaluation job needs a small data contract, not a privileged DNS API. That matters when the zone no longer belongs to the registrar that used to host it.

## Why a record check cannot certify deliverability

A DNS response says that one resolver obtained data. It does not prove that a receiving system accepted a message, aligned the relevant identifiers, or applied the published policy as the sender expected. DMARC itself is evaluated by receivers, and RFC 7489 makes clear that receivers use DNS-published policy in their handling. Those are separate stages.

The record and the outcome answer different questions.

The practical trade-off is latency against confidence. DNS checks detect a malformed or absent record quickly and without generating mail. Outcome checks take longer, require a controlled recipient and a way to capture authentication results, but test the path the business actually depends on. For lease-renewal reminders, I would alert on missing DNS evidence immediately, then promote the incident only when the 2-hour outcome window remains negative after the cache horizon chosen for that domain.

A single successful probe is weak evidence. Recipient policy, routing, and cached state vary. Keep the probe identity stable, use a unique message token per run, and record an explicit `unknown` state when outcome instrumentation is absent. Treating unknown as delivered makes dashboards pleasant and operations brittle.

## The operational rule I would automate

Start each migration with an expected-record manifest: domain, record owner, required value fragment, resolver classes, and the intended probe mailbox. Run DNS observations before and after the change, then correlate only outcomes whose message tokens fall inside the post-change window. Keep the old zone's expiry and the new observation timestamps beside the incident record; they explain most apparent contradictions without guessing.

When the record is present but probes fail, inspect the captured authentication result and message trace before editing DNS again. When several resolver classes miss the record, check delegation and authoritative data. When only one resolver class is behind, let its cache age out according to the evidence you captured and continue measuring. This sequence avoids the common mistake of rewriting a correct policy because one cache has not caught up.

The decision is deliberately conservative: mark the migration healthy only after the expected DNS evidence and authenticated delivery evidence agree. Everything else is a state worth observing.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc2308
