# Bulk Password Reset Notifications: Email and SMS Queue Worker Deadlines

TL;DR: For bulk password-reset notifications, enqueue one logical delivery per recipient, then let workers claim small batches and re-check expiry immediately before each email or SMS attempt. Cron polling is useful as a recovery trigger, not as the clock that defines correctness. The decisive metric is useful delivery before the reset token expires; a late “successful” send is still a product failure.

That changes the implementation. A queue worker should spend a limited time budget on retries, suppress work that has become stale, and record channel attempts separately from the reset request. Email authentication and SMS segmentation matter, but neither can rescue a message that sat behind a large batch until its link was already useless.

## Why can a successful batch still fail the user?

A bulk API reports acceptance, not timely completion of the password reset. Email may pass through several relays after acceptance. SMS text can be split into multiple segments: GSM-7 messages use a different single-message and concatenated-segment capacity from UCS-2 messages, so one non-GSM character can change the number of segments. That is a delivery concern because more segments create more pieces that must reach and be reassembled for the recipient.

DKIM answers a different question. RFC 6376 defines a domain-level signature that lets a verifier associate a message with a signing domain and detect modification of signed content in transit. It does not promise inbox placement, receipt, or arrival before an application deadline. **Provider acceptance, authenticated mail, and useful delivery are three separate states.**

For a concrete policy, suppose the application chooses a 10-minute reset lifetime. That number is an application decision in this example, not a universal recommendation. The request row stores an absolute `expires_at`; each queued delivery copies that deadline. Workers use it to refuse late sends, while the reset endpoint remains the authority that rejects an expired token.

No guesswork here: the notification system never extends token validity.

Deadlines win.

## Put the deadline in the job contract

The data flow is short. The application creates the reset request and an outbox record in the same database transaction. A dispatcher turns undispatched outbox rows into delivery jobs. Workers claim bounded batches, call a channel adapter, and save an attempt result. A periodic poller revisits eligible jobs that were left behind by a stopped worker or a transient channel failure.

The job needs an identity that survives retries. A useful key is `(reset_request_id, recipient_id, channel)`, enforced by a unique database constraint. Do not generate a fresh logical ID every time cron finds the row; that converts recovery into duplication. The payload should contain a template identifier and data, not a rendered token-bearing message sitting indefinitely in a broker.

This runnable example keeps storage in memory so the state machine is visible. Production code should implement the same compare-and-set claim and unique key in a durable database.

```python
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Callable
from uuid import UUID, uuid4


class State(str, Enum):
    READY = "ready"
    LEASED = "leased"
    RETRY = "retry"
    SENT = "sent"
    EXPIRED = "expired"


@dataclass
class Delivery:
    reset_id: UUID
    recipient: str
    channel: str
    expires_at: datetime
    id: UUID = field(default_factory=uuid4)
    state: State = State.READY
    attempts: int = 0
    next_attempt_at: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    lease_until: datetime | None = None


def claim_batch(
    jobs: list[Delivery], now: datetime, limit: int, lease: timedelta
) -> list[Delivery]:
    claimed: list[Delivery] = []
    for job in sorted(jobs, key=lambda item: item.next_attempt_at):
        lease_expired = job.state == State.LEASED and job.lease_until <= now
        eligible = job.state in {State.READY, State.RETRY} or lease_expired
        if eligible and job.next_attempt_at <= now and len(claimed) < limit:
            job.state = State.LEASED
            job.lease_until = now + lease
            claimed.append(job)
    return claimed


def process(
    job: Delivery,
    now: datetime,
    send: Callable[[Delivery], None],
) -> None:
    if now >= job.expires_at:
        job.state = State.EXPIRED
        job.lease_until = None
        return

    try:
        send(job)
    except TimeoutError:
        job.attempts += 1
        delay = timedelta(seconds=min(30 * (2 ** (job.attempts - 1)), 120))
        next_try = now + delay
        job.state = State.RETRY if next_try < job.expires_at else State.EXPIRED
        job.next_attempt_at = next_try
        job.lease_until = None
        return

    job.attempts += 1
    job.state = State.SENT
    job.lease_until = None


def run_once(
    jobs: list[Delivery], now: datetime, send: Callable[[Delivery], None]
) -> None:
    for job in claim_batch(jobs, now, limit=25, lease=timedelta(seconds=30)):
        process(job, now, send)
```

The adapter represented by `send` should translate a channel response into a small internal taxonomy: accepted, permanent rejection, or retryable uncertainty. A timeout is uncertain, not proof that nothing happened. That is why the stable delivery ID should also be passed to an adapter that supports deduplication, and why attempt records must retain the provider's correlation identifier when one is returned.

There is a deliberate trade-off in the example. It retries only while the next attempt still fits before `expires_at`. More aggressive retries may raise the chance of a timely send, but they also increase load during an outage and can crowd out newer reset requests. I would tune the delay and batch size from queue-age percentiles and deadline misses, not from an arbitrary “messages per worker” target.

## How should a queue worker send bulk email and SMS event notifications?

Batching reduces call and database overhead, yet a giant batch creates head-of-line delay. Claim 25 jobs, as the example does, and that is an explicit starting assumption to evaluate, not a magic value. The eval harness should replay bursts with mixed email and SMS latency, inject timeouts after acceptance, stop a worker after its claim, and advance the clock across expiry. Assertions belong on state transitions: no send begins at or after expiry, a reclaimed lease does not create a second logical job, and every terminal job is either sent, permanently rejected, or expired. I prefer those invariant checks over snapshots of adapter output because they survive template changes and expose the race that matters: a job can be valid when claimed, spend 30 seconds behind other calls, and be stale by the time its adapter runs. Test that boundary with a fake clock, then repeat it with the first call timing out after acceptance. The second execution must retain the same logical delivery identity even though the attempt identity changes.

Retries aren't free.

Keep email and SMS concurrency limits independent. Their payload constraints and downstream behavior differ, and one impaired channel should not consume every worker slot. Channel fallback also needs a policy decision. Sending SMS immediately after an ambiguous email timeout can produce two messages; waiting may consume the short expiry window. For password resets, a clean design chooses the preferred channel when the request is created and uses fallback only with explicit consent and a remaining-time threshold.

Prompt cost belongs outside this hot path. A password-reset notification should use a reviewed deterministic template; generating security copy during dispatch adds latency, variability, and another failure dependency. If an AI feature selects locale or tone elsewhere, store the resulting bounded template choice before enqueueing and evaluate it offline against approved outputs.

Keep generation elsewhere.

## What should cron polling repair?

Cron should wake a reconciler frequently enough for the chosen deadline, query only indexed rows whose `next_attempt_at` is due, and make expired leases eligible again. It should not blindly resend every non-terminal row. Multiple reconciler instances may overlap, so the database claim must be atomic; the in-memory loop above only illustrates the rule.

Use the same worker function for queue-triggered and poll-triggered jobs. Two execution paths with different expiry or retry checks are hard to test and eventually disagree. The broker can provide fast wake-ups, while the database provides the durable ledger and the poller repairs missed wake-ups.

Observe useful outcomes. Track age at claim, age at acceptance, expired-before-send count, retry reason, lease recovery count, and terminal state by channel. Avoid putting reset tokens, email addresses, or phone numbers into logs and metric labels. A hashed or opaque delivery ID is enough to join traces to the protected operational record.

## Ship the worker with a deadline-focused check

Before deployment, run a deterministic clock through the state machine and prove behavior one second before and exactly at expiry. Exercise ASCII SMS copy and copy containing non-GSM characters so segmentation is visible before production. Verify the email signer against the DKIM specification, then test that rendering changes do not invalidate the signed content. Load the queue with a burst larger than one claim batch, pause a worker until its lease ends, and confirm that recovery preserves the logical delivery ID.

Finally, review the operational knobs together: reset lifetime, maximum retry delay, lease duration, batch size, and per-channel concurrency. They form one time budget. **The worker is ready when it protects that budget under retries and recovery, not when it can merely drain a happy-path batch.**

## Sources

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
