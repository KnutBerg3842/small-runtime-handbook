# Missed Freight Updates: A Realtime API Approach Using Sequence Numbers

TL;DR: Give every freight-auction event a monotonically increasing sequence number within a named scope, persist the consumer's last applied number, and reconnect by requesting events after that cursor before resuming the live stream. For a small SaaS team, this is the least complex design that makes missed in-app notifications detectable without polling. Treat the stream as a low-latency hint and replay as the recovery authority.

A carrier sees bid 184, loses connectivity while bids 185 and 186 arrive, then reconnects as bid 187 is broadcast. Without an ordered cursor, the screen can look current while silently omitting the winning change. With one, the client can prove there is a gap and backfill it.

## How should a realtime messages API use sequence numbers?

The client cannot infer completeness from a healthy connection. It needs an application invariant: within one auction, the next accepted event has the previous sequence plus one. A jump from 184 to 187 is evidence that 185 and 186 must be fetched; a repeated 184 is safe to ignore after it has been applied.

The gap is visible.

Use a cursor such as `(auction_id, sequence)`, not a process-local counter. Allocate it in the same durable write boundary as the bid event, then expose a read operation equivalent to `events after sequence 184 for auction A`. The transport may be WebSocket, Server-Sent Events, long polling, or a data channel. None creates this application invariant.

**The key selection criterion is recoverability, not connection uptime.** Verify that an approach can return an ordered, retained slice after a supplied cursor and that retention exceeds the promised reconnect interval.

## Build the cursor contract first

This runnable model separates receiving from applying, preventing event 187 from reaching the view before backfill supplies 185 and 186. The contract maps directly to a Node.js service, while Python keeps the notebook-to-production recovery test compact.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Bid:
    auction: str
    sequence: int
    cents: int

class View:
    def __init__(self, auction: str):
        self.auction, self.last, self.cents = auction, 0, 0

    def apply(self, event: Bid) -> str:
        if event.auction != self.auction:
            raise ValueError("wrong auction")
        if event.sequence <= self.last:
            return "duplicate"
        if event.sequence != self.last + 1:
            return "gap"
        self.last, self.cents = event.sequence, event.cents
        return "applied"

def recover(view: View, live: Bid, retained: list[Bid]) -> None:
    if view.apply(live) != "gap":
        return
    missing = [e for e in retained if view.last < e.sequence <= live.sequence]
    for event in sorted(missing, key=lambda e: e.sequence):
        if view.apply(event) == "gap":
            raise RuntimeError("retention cannot fill gap")

history = [Bid("lane-42", 1, 128_000), Bid("lane-42", 2, 126_500), Bid("lane-42", 3, 125_900)]
view = View("lane-42")
recover(view, history[0], history)
recover(view, history[2], history)
assert (view.last, view.cents) == (3, 125_900)
```

Contiguous integers make gaps visible. They do not order different auctions and should not encode wall-clock time. An opaque cursor can support replay too, but then the server must declare replay complete because the client cannot diagnose a missing integer.

A bid can be committed yet never published when those are separate writes. A transactional outbox is one boundary to evaluate: record the domain change and outgoing event together, then let a relay retry. This moves complexity into retention, relay monitoring, and cleanup, but makes recovery inspectable.

There is a real limitation: strict per-auction sequencing adds coordination to the write path. It is a poor fit when every message is disposable, clients never reconstruct state, or the workload requires unrelated producers to share one global order. In those cases, independent streams or an opaque replay cursor may be enough. For freight bids, where one omitted update can leave the displayed leader and amount stale, the extra contract earns its keep. A small team should still avoid a global sequence across all auctions; scoped counters contain contention and make ownership clearer.

## Reconnect is a merge

On reconnect, buffer live traffic, fetch everything after the persisted cursor, apply replay in order, and drain the buffer through the same deduplicating function. Only then enter live mode. If the cursor predates retained history, fetch an authoritative snapshot paired with its cursor.

This closes a race in which events arrive after the replay query reads its upper bound but before the subscription is usable. Explicit states such as `connecting`, `backfilling`, `live`, and `resync_required` are more informative than a green connection badge.

WebRTC data channels show why transport and recovery remain separate. The standard defines ordered delivery as the default and permits unordered operation, with retransmission controls. Those properties govern an existing channel. A fresh connection still needs application state describing what was applied and where replay starts.

Test with generated histories, not sleeps. Cut after every possible sequence, duplicate boundary events, deliver the first post-reconnect event early, and assert that the final projection equals a clean fold of the ordered log. Track missing sequences, duplicate applications, and divergence from the authoritative snapshot. This eval-driven habit exposes the contract before transport details obscure it.

## Decide with failure evidence

Compare candidates using one fixture: a known auction history, deterministic disconnects, duplicates, and a cursor beyond retention. Record whether the client converges, how an unrecoverable cursor is signaled, and which component owns replay.

Automatic reconnection is insufficient evidence. It can restore a socket while leaving the view incomplete. Persist the cursor after the projection update, not before it, and distinguish receipt from application.

No guesswork required.

Measure gap detections, replay counts, replay age, duplicate drops, resync requests, and time spent backfilling. Avoid bid amounts and customer details in logs. Alert on sustained failure to converge rather than every routine reconnect.

Before release, run the deterministic history suite in CI and a disconnect drill in staging. Confirm retained history covers the declared offline window, snapshot fallback returns state with a cursor, and one trace connects disconnect, replay, and live mode. Choose the implementation that upholds this contract and proves convergence under forced gaps. Sequence numbers turn silent omission into a testable transition.

## Further reading

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- MDN, Using server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- MDN, WebSocket API: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- AWS Prescriptive Guidance, Transactional outbox pattern: https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html
