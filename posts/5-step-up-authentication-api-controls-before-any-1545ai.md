# 5 Step-Up Authentication API Controls (Before Any Sensitive Action)

TL;DR: Keep ordinary sign-in separate from authorization for a sensitive action. Before a fintech app changes a payout destination, adds a beneficiary, or reveals account data, require a fresh phone code and exchange that proof for a short-lived, single-use ticket bound to the user, session, and exact action. Put this contract inside the app rather than inside a managed provider's session format. That is the least complex design that supports step-up verification now and a provider migration later.

The data flow is small. The authenticated client asks to verify one named action, the server sends a one-time code through the current delivery adapter, and a successful check issues an opaque step-up ticket. The sensitive endpoint consumes that ticket atomically. A provider can deliver or verify the phone code, but it does not decide what `change_payout_account` means or authorize the money-moving request.

## 1. How should an API step up authentication before a sensitive action?

Accept a server-controlled capability, not a Boolean such as `recently_verified: true` supplied by the client. The capability needs a narrow subject and purpose: user ID, current session ID, action type, target fingerprint, expiration time, and a unique identifier. The endpoint checks every field and marks the identifier used in the same transaction as the protected change.

This distinction matters during migration. A managed provider's login token can continue proving who signed in while the application owns the smaller claim: this person freshly verified this particular operation. If phone delivery moves later, the authorization contract stays put.

Keep that boundary boring.

OWASP recommends reauthentication for sensitive features and after risk events, then invalidating sessions and rotating tokens as appropriate. It also warns that sensitive transaction authorization should not rely only on an active session. Those recommendations point to an action-scoped proof rather than a blanket extension of the login session.

The target fingerprint closes a subtle gap. A ticket requested for payout account A must fail if the final request names payout account B. Hash a canonical representation of the target fields on the server; never trust a digest computed only by the browser. For an action without a target, bind an explicit constant such as `view_tax_document`.

The trade-off is extra state and another server-side check on the critical path. This pattern is a poor fit for a low-risk, read-only action where ordinary session authorization already matches the harm: adding a challenge there creates abandonment without narrowing meaningful risk. It is also incomplete for approvals that legally or operationally require two people, hardware-backed credentials, or transaction signing. Use the stronger control demanded by that workflow; a phone code plus an application ticket cannot manufacture those properties.

## 2. Build the verifier as a narrow application contract

The following Python example models the core boundary without a commercial SDK. It signs a compact ticket, verifies its scope, and delegates single-use enforcement to a store. The in-memory store makes the example runnable; production code needs a shared store whose consume operation is atomic across application instances.

```python
import base64
import hashlib
import hmac
import json
import time
import uuid
from dataclasses import dataclass, field


def encode(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")


def decode(value: str) -> bytes:
    return base64.urlsafe_b64decode(value + "=" * (-len(value) % 4))


def target_fingerprint(target: dict[str, str]) -> str:
    canonical = json.dumps(target, sort_keys=True, separators=(",", ":"))
    return hashlib.sha256(canonical.encode()).hexdigest()


@dataclass
class OneTimeStore:
    used: set[str] = field(default_factory=set)

    def consume(self, ticket_id: str) -> bool:
        if ticket_id in self.used:
            return False
        self.used.add(ticket_id)
        return True


def issue_ticket(
    secret: bytes,
    user_id: str,
    session_id: str,
    action: str,
    target: dict[str, str],
    now: int,
    lifetime_seconds: int = 300,
) -> str:
    claims = {
        "sub": user_id,
        "sid": session_id,
        "act": action,
        "target": target_fingerprint(target),
        "iat": now,
        "exp": now + lifetime_seconds,
        "jti": str(uuid.uuid4()),
    }
    payload = encode(json.dumps(claims, separators=(",", ":")).encode())
    signature = encode(hmac.new(secret, payload.encode(), hashlib.sha256).digest())
    return f"{payload}.{signature}"


def consume_ticket(
    token: str,
    secret: bytes,
    store: OneTimeStore,
    expected_user_id: str,
    expected_session_id: str,
    expected_action: str,
    expected_target: dict[str, str],
    now: int,
) -> bool:
    try:
        payload, supplied_signature = token.split(".", 1)
        expected_signature = encode(
            hmac.new(secret, payload.encode(), hashlib.sha256).digest()
        )
        if not hmac.compare_digest(supplied_signature, expected_signature):
            return False
        claims = json.loads(decode(payload))
    except (ValueError, json.JSONDecodeError):
        return False

    correct_scope = all(
        (
            claims.get("sub") == expected_user_id,
            claims.get("sid") == expected_session_id,
            claims.get("act") == expected_action,
            claims.get("target") == target_fingerprint(expected_target),
            isinstance(claims.get("iat"), int),
            isinstance(claims.get("exp"), int),
            claims.get("iat") <= now < claims.get("exp"),
            isinstance(claims.get("jti"), str),
        )
    )
    return correct_scope and store.consume(claims["jti"])


if __name__ == "__main__":
    key = b"replace-with-a-secret-from-your-secret-manager"
    clock = int(time.time())
    payout_target = {"routing": "021000021", "account_suffix": "8842"}
    ticket = issue_ticket(
        key, "user_1042", "session_7f3", "change_payout_account",
        payout_target, clock
    )
    tickets = OneTimeStore()
    assert consume_ticket(
        ticket, key, tickets, "user_1042", "session_7f3",
        "change_payout_account", payout_target, clock
    )
    assert not consume_ticket(
        ticket, key, tickets, "user_1042", "session_7f3",
        "change_payout_account", payout_target, clock
    )
```

Five minutes in this example is a policy choice, not a universal security constant. Set the lifetime from the time a person reasonably needs to finish the operation, then test the boundary at `exp - 1`, `exp`, and after expiration. A clock you can inject makes that evaluation deterministic. Keep code-entry attempts and resend limits in the phone-code component; keep ticket consumption in the sensitive-action component.

Three boundary checks. No guesswork.

Do not log the code, the ticket, or full payout details. Log stable event names and non-secret correlation IDs instead. Fast notebook experiments often print entire dictionaries, and that habit becomes dangerous when copied into a production handler.

## 3. Separate phone-code proof from action authorization

Phone possession is one verification signal. It is not a durable declaration that every later request is safe. A useful boundary has two operations: verify the challenge, then issue the application ticket. The first can sit behind an adapter for a managed service or an internal component; the second belongs to the application's authorization policy.

| Decision | Phone-code component | Sensitive-action component |
| --- | --- | --- |
| Was the challenge answered? | Yes | Reads verified evidence |
| Does the user have permission? | No | Yes |
| Does the approved target match? | No | Yes |
| Can this approval be replayed? | Invalidates the challenge | Atomically consumes the ticket |

**Treat delivery success, code verification, and action authorization as three different outcomes.** A message accepted for delivery does not prove receipt. A correct code does not prove that the requested payout target matches the one shown before verification. A valid ticket does not excuse the endpoint from checking the user's ordinary permission to perform the action.

Store one-way protected code material rather than a reusable plaintext code, enforce expiration and limited attempts on the server, and make successful verification single-use. Responses should avoid revealing whether a phone number belongs to an account. OWASP's guidance calls for generic authentication responses because differing messages or response behavior can enable account enumeration.

There is also a recovery boundary. If the phone number itself is being replaced, verification through that same number should not silently establish the new number as trusted. Recovery and factor replacement need their own policy, user notification, and session handling. Short code. Big distinction.

## 4. Migrate with shadow evaluation, not dual authority

During a provider move, avoid a period where either of two systems can independently bless the sensitive action. Choose one application ticket issuer as the authority. The old and new phone-verification paths may run behind an adapter, but only the active path can produce accepted verification evidence.

Shadow evaluation is still valuable. Feed sanitized challenge metadata to the candidate path, suppress external delivery, and compare decisions in an evaluation harness. Track agreement by reason category: expired challenge, wrong code, attempt limit, already consumed, and mismatched user or session. Raw codes and phone numbers do not belong in that dataset.

The cutover rule should be behavioral, not SDK-shaped. Consider one concrete migration case: a user requests a code for account suffix `8842`, requests a resend, enters the first code, and then submits two payout-change requests at nearly the same time. The first code must be invalid after resend, and no more than one request using the eventual ticket may succeed. Next, replace the login session before ticket consumption; the old ticket must fail because its session binding no longer matches. Repeat with a changed suffix, an exact expiration boundary, a delivery timeout, and a duplicate verification callback. A candidate is ready when those contract cases pass consistently. This is where notebook-to-production discipline pays off: promote the fixed corpus into CI, then run the same cases against every adapter.

Prompt cost is irrelevant to the authorization decision itself, but it matters if an AI model classifies support notes or risk context around the flow. Keep model output advisory. Record its version and input class for evaluation, cap its token budget, and never let generated text mint a step-up ticket. A deterministic policy must remain the final gate for moving money or exposing financial records.

**One authority at a time** also makes rollback comprehensible. Switch the verification adapter back while leaving ticket semantics, endpoint checks, audit fields, and expiration policy unchanged. Migration then changes a dependency, not the meaning of authorization.

Rollback stays small.

## 5. Operate the boundary as a security control

Instrument the funnel without collecting secrets: challenge requested, dispatch accepted, verification passed or rejected by coarse reason, ticket issued, ticket consumed, and sensitive action completed. Measure abandonment and latency by client version and action. Alert on abrupt changes in resend volume, rejection categories, or ticket replay; investigate before loosening a control.

The operational checklist is prose because the dependencies matter. Start by inventorying every sensitive action and assigning an exact action name. Define what target data must be bound for each one, who may invoke it, and which risk events force fresh verification. Confirm that ticket issuance happens only after server-side code verification. Confirm that consumption is atomic, expiration uses server time, session termination invalidates outstanding authority, and logs exclude codes and tokens. Then exercise concurrent requests and failure injection in a staging environment before moving traffic.

Finally, review the user-visible path. The screen should name the action being approved, mask the destination for the code, provide a controlled resend path, and return a generic error when verification fails. Security teams get a narrow audit trail; support teams get correlation IDs; users get a clear reason for the interruption without learning internal risk rules.

The decision is straightforward: own the action-scoped contract, keep phone verification replaceable, and require both normal authorization and fresh proof at the sensitive endpoint. That structure limits replay, prevents target swapping, and lets a fintech team change delivery infrastructure without redefining what approval means.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
