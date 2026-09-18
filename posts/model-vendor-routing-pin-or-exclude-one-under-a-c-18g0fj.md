# Model Vendor Routing: Pin or Exclude One (Under a Clinical Spend Ceiling)

Short answer: pin each healthtech tenant to a small model vendor allowlist; exclude one only as a temporary routing control, and fail closed when the spend ceiling or key scope is uncertain. The deciding constraint is not abstract flexibility. It is whether the API should refuse some clinical-assistant traffic rather than send a request through an unapproved provider or an unbounded credential.

That answer is deliberately uncomfortable. An allowlist can reject traffic during a provider outage, while an exclusion rule may keep more requests moving. For a tenant-scoped clinical workflow, though, the permissive fallback silently expands the set of destinations and credentials that can consume budget. I would rather make refusal explicit, observable, and testable.

The experiment I would run before adopting this policy is simple: replay the same evaluation corpus through provider failure, key revocation, and spend-ceiling events. Score task quality as usual, but add policy outcomes: permitted, refused, and incorrectly routed. A quality score alone cannot reveal that a good answer traveled through a destination the tenant never approved.

## Should routing pin one model vendor or exclude it?

An exclusion rule describes the one place traffic must not go. Everything else is implicitly eligible. That feels adaptable when the model vendor catalog is small, and it is tempting in a notebook because a single string filter gets the demo moving. The meaning changes as soon as a new destination appears in discovery: it can become eligible without the tenant making a new decision. This is why exclusion ages worse than an explicit pin for a sensitive API.

An allowlist reverses that default. A newly discovered provider remains unavailable until the tenant policy names it. This creates refused traffic during some outages, but it keeps eligibility stable as the surrounding catalog changes. That is the property I care about for a long-lived API.

I initially expect exclusions to age better in short notebook experiments because they preserve optionality. Then I write the failure test, and the attraction fades. If provider A is denied after an incident, an unconstrained fallback to provider C might pass the model-quality eval while violating the tenant's destination policy or using a credential with a different ceiling. The answer can look correct. The route is still wrong.

The comparison is compact:

| Constraint | New provider appears | Approved provider fails | Best operational role |
| --- | --- | --- | --- |
| Tenant allowlist | Refused until approved | Refused unless another approved route fits | Durable policy boundary |
| Provider denylist | Eligible unless separately blocked | Can continue through any remaining route | Time-bounded incident containment |

This is not an argument for one permanent provider. A tenant can allow two or three destinations and rank them by evaluated fitness. The durable choice is the closed set, not the size of the set. Its limitation is plain: if every approved destination is unavailable, the request is refused even when an unapproved vendor could answer it. Teams that value uninterrupted traffic above destination control should not use this fail-closed policy; they need an explicitly open fallback policy and must accept its wider authorization boundary.

## Put the tenant boundary before model selection

Routing should be an intersection, not a preference list. Start with the tenant's active scoped key, intersect its allowed providers with the providers that remain under the tenant's spend ceiling, and only then apply latency or model-quality preferences. If the intersection is empty, refuse the request.

Keep key identity separate from the secret value. The policy can carry an opaque key identifier, tenant identifier, scopes, status, and ceiling state; a secrets manager resolves the secret only after authorization. OWASP's Secrets Management guidance recommends least privilege, automated rotation, revocation, auditing, and avoiding secrets in logs. Those practices fit this boundary directly: routing metadata may be observable, while the credential must not be.

A focused Python policy object makes the order hard to misread:

```python
from dataclasses import dataclass
from decimal import Decimal
from enum import Enum


class Decision(str, Enum):
    PERMIT = "permit"
    REFUSE = "refuse"


@dataclass(frozen=True)
class TenantPolicy:
    tenant_id: str
    key_id: str
    key_active: bool
    scopes: frozenset[str]
    allowed_providers: frozenset[str]
    spend_ceiling: Decimal
    committed_spend: Decimal


def authorize_route(
    policy: TenantPolicy,
    provider: str,
    required_scope: str,
    reserved_cost: Decimal,
) -> Decision:
    if not policy.key_active:
        return Decision.REFUSE
    if required_scope not in policy.scopes:
        return Decision.REFUSE
    if provider not in policy.allowed_providers:
        return Decision.REFUSE
    if reserved_cost < Decimal("0"):
        return Decision.REFUSE
    if policy.committed_spend + reserved_cost > policy.spend_ceiling:
        return Decision.REFUSE
    return Decision.PERMIT
```

There are two intentional details. Money uses `Decimal`, so the policy does not smuggle binary floating-point behavior into a boundary check. More important, the function accepts reserved cost rather than waiting for a final invoice. A ceiling evaluated only after completion is accounting, not admission control. The caller must obtain a conservative reservation using its own tokenizer, model limits, and contract data; if it cannot, the safe result is refusal.

Short is useful here.

The router may choose among the providers that return `PERMIT`, but it must not reinterpret `REFUSE` as a hint to broaden the set. That separation keeps an optimization change from becoming an authorization change.

## Issue and revoke one scoped key per tenant

A tenant key should have one lifecycle record even when several approved provider credentials sit behind it. Issuance creates an opaque identifier, a minimal scope set, an allowed-provider set, a spend ceiling, and an audit event. The returned secret is shown once and stored hashed or otherwise protected according to the key design; downstream provider secrets belong in a secrets manager, not in the tenant record or application logs.

Revocation flips the tenant key to inactive before any route selection. Propagate that state to every router instance and invalidate cached authorization decisions. Provider credential rotation is a separate operation: change the resolved secret behind its identifier without quietly changing which providers the tenant permits. Combining those actions makes incident response harder to reason about because a credential repair can accidentally alter routing policy.

Race conditions deserve a specific test. Suppose two requests each fit beneath the remaining ceiling when read independently, but their combined reservation does not. A read-then-write check admits both. Use an atomic reservation in the budget ledger, attach an idempotency key, and release unused reservation after the provider reports usage. If the reservation cannot be recorded, refuse before making the external call.

Do not log the key. Log the tenant identifier, opaque key identifier, policy version, candidate provider, decision reason, reservation identifier, and trace identifier. This is enough to reconstruct why traffic was refused without putting reusable credentials into an observability system.

## Test the refusals, not merely the happy route

My notebook-to-production checkpoint is a table-driven policy test before any live routing test. It has no network dependency and no model variance, so a regression means the policy changed.

```python
from decimal import Decimal


def test_route_policy_matrix() -> None:
    base = TenantPolicy(
        tenant_id="clinic-017",
        key_id="key-042",
        key_active=True,
        scopes=frozenset({"clinical-summary"}),
        allowed_providers=frozenset({"provider-a", "provider-b"}),
        spend_ceiling=Decimal("500.00"),
        committed_spend=Decimal("498.50"),
    )

    cases = [
        ("provider-a", "clinical-summary", "1.25", Decision.PERMIT),
        ("provider-c", "clinical-summary", "1.25", Decision.REFUSE),
        ("provider-a", "bulk-export", "1.25", Decision.REFUSE),
        ("provider-b", "clinical-summary", "2.00", Decision.REFUSE),
    ]

    for provider, scope, reservation, expected in cases:
        actual = authorize_route(base, provider, scope, Decimal(reservation))
        assert actual is expected
```

Add property tests around the invariant: adding an unknown provider to discovery cannot turn a refusal into a permit. Then run replay evaluations that combine this policy layer with task quality. At minimum, inject an inactive tenant key, a removed provider, a stale policy version, an exhausted ceiling, a reservation collision, and an unavailable approved provider.

The metric I want first is `incorrect_route_total`, partitioned by policy version and decision reason. It should remain zero. Refusal rate comes next, because a high value can expose ceilings that are too tight or an approved set that lacks resilience. Latency and answer-quality metrics matter after authorization is correct; optimizing them earlier rewards the router for escaping the constraint.

Watch prompt cost as an input to admission, too. A prompt expansion can increase the conservative reservation even when request count is flat. Track reserved versus reported usage by model class, without logging clinical content, and alert on widening estimation error. This is where an eval harness earns its place outside the notebook: it connects prompt changes to both output quality and admission behavior.

## Use deny rules as expiring incident controls

A deny rule still has a job. It can remove one currently allowed destination quickly during a security or compliance event, but it should narrow an existing allowlist rather than replace it. Give the rule an owner, reason, creation time, expiration time, and review state. An expired rule should trigger review; it must not silently become permanent architecture.

The effective set is therefore `tenant allowlist - active incident denies`. This composition gives responders a fast brake while preserving the tenant's closed-world boundary. If the result is empty, return a stable refusal category that callers can handle without retrying across unapproved destinations.

Before copying this choice, measure three things in a replay: the share of traffic refused when each approved provider is unavailable, the count of routes that violate the tenant policy, and the reservation error between admitted and reported usage. The spend ceiling versus refused-traffic trade-off becomes visible in those numbers. Choose the allowlist and ceiling only after the team agrees which failures may degrade, which must refuse, and who can change that policy.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
