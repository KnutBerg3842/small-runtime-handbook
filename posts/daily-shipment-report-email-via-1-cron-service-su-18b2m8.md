# Daily Shipment Report Email via 1 Cron Service (Subscriber Delivery Guarantees)

Short answer: use the cron service only to trigger one authenticated public webhook, then let a durable outbox fan the daily shipment report out to subscribers with idempotency, bounded retries, and a dead-letter path. The easiest setup is a thin endpoint; the reliable system begins after that endpoint accepts the trigger.

For a logistics team, the useful unit is not "the cron ran." It is "every eligible subscriber got one report for the intended reporting day, or an operator can explain why not." A scheduler calls the callback once with a stable run key such as `shipment-report:2026-08-20`. The callback records that run and its recipient jobs in one transaction, returns quickly, and leaves email delivery to workers. This keeps a slow provider or a large subscriber list away from the scheduler's timeout boundary.

Keep it boring.

## Security boundary: authenticate before accepting a run

The example below uses Python and SQLite so the delivery state is visible instead of hidden behind framework helpers. The same boundary fits a Node.js Express handler: authenticate the request, claim an idempotency key, enqueue one job per subscriber, and return `202` without sending email inline. The scheduler needs a public HTTPS endpoint and a secret header; the application needs a durable database.

```python
import hashlib
import hmac
import json
import os
import sqlite3
from datetime import datetime, timezone
from http.server import BaseHTTPRequestHandler, HTTPServer

DB_PATH = os.environ.get("REPORT_DB", "reports.db")
WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]


def connect():
    db = sqlite3.connect(DB_PATH)
    db.execute("PRAGMA journal_mode=WAL")
    db.executescript("""
        CREATE TABLE IF NOT EXISTS report_runs (
            run_key TEXT PRIMARY KEY,
            report_date TEXT NOT NULL,
            accepted_at TEXT NOT NULL
        );
        CREATE TABLE IF NOT EXISTS subscribers (
            subscriber_id TEXT PRIMARY KEY,
            email TEXT NOT NULL,
            active INTEGER NOT NULL DEFAULT 1
        );
        CREATE TABLE IF NOT EXISTS email_outbox (
            run_key TEXT NOT NULL,
            subscriber_id TEXT NOT NULL,
            state TEXT NOT NULL DEFAULT 'pending',
            attempts INTEGER NOT NULL DEFAULT 0,
            next_attempt_at TEXT NOT NULL,
            PRIMARY KEY (run_key, subscriber_id),
            FOREIGN KEY (run_key) REFERENCES report_runs(run_key)
        );
    """)
    return db


def valid_signature(body, supplied):
    expected = hmac.new(
        WEBHOOK_SECRET.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, supplied)


def accept_run(payload):
    run_key = payload["run_key"]
    report_date = payload["report_date"]
    now = datetime.now(timezone.utc).isoformat()

    with connect() as db:
        inserted = db.execute(
            "INSERT OR IGNORE INTO report_runs VALUES (?, ?, ?)",
            (run_key, report_date, now),
        ).rowcount
        if inserted:
            db.execute("""
                INSERT OR IGNORE INTO email_outbox
                    (run_key, subscriber_id, next_attempt_at)
                SELECT ?, subscriber_id, ?
                FROM subscribers WHERE active = 1
            """, (run_key, now))
    return inserted == 1


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/hooks/daily-shipment-report":
            self.send_error(404)
            return

        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length)
        signature = self.headers.get("X-Webhook-Signature", "")
        if not valid_signature(body, signature):
            self.send_error(401)
            return

        try:
            payload = json.loads(body)
            created = accept_run(payload)
        except (KeyError, json.JSONDecodeError):
            self.send_error(400)
            return

        response = json.dumps({
            "accepted": True,
            "duplicate": not created,
            "run_key": payload["run_key"],
        }).encode()
        self.send_response(202)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(response)))
        self.end_headers()
        self.wfile.write(response)


if __name__ == "__main__":
    connect().close()
    HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

Run it with a secret supplied by the deployment environment. A cron service can compute the HMAC over its JSON request body and send that digest in `X-Webhook-Signature`; if the chosen scheduler cannot set a dynamic signature, put an authenticated gateway in front rather than placing the secret in a query string. The payload should be small and deterministic: `run_key`, `report_date`, and perhaps a schema version. Subscriber addresses and report content remain server-side.

```bash
WEBHOOK_SECRET=replace-with-a-secret python report_hook.py
```

There is a deliberate constraint here: the callback snapshots active subscriber IDs, not the full email body. A worker can render from report data later, while the composite primary key prevents the same run from producing a second job for the same subscriber. If membership must reflect the exact scheduled instant, snapshot every required personalization field in the transaction too. That's more storage, but it removes ambiguity when a subscriber changes preferences during delivery.

## How should a daily report email cron service call a public webhook endpoint?

Treat the call as an at-least-once trigger, even if a scheduler describes its execution in friendlier terms. Networks can lose the response after the server commits, so a retry can be indistinguishable from a new request unless both attempts carry the same `run_key`. Returning `202` means the run was durably accepted; it does not claim that every email has already left the system. A duplicate request should return the same successful class of response and create no additional outbox rows.

The schedule also needs an explicit time zone and reporting interval. "Every day at 09:00" is incomplete around daylight-saving changes and for warehouses spread across regions. Define the report by a half-open interval such as `[start, end)` in UTC, derive the idempotency key from the business date plus report type, and store the interval with the run. I'm not sure a single business date is sufficient for every logistics network; the deciding evidence is whether late scans are assigned by event time, ingestion time, or local depot time. Write that policy before choosing the cron expression.

The catch is that a public callback is not suitable when the application cannot accept inbound traffic or when the organization requires every scheduled action to remain inside a private network. In that case, keep the schedule beside a queue in the private environment, or run a resident scheduler with leader election. A resident scheduler is also a reasonable choice for sub-minute coordination. Stick with the public callback pattern when one coarse daily trigger, easy independent replacement, and a narrow security boundary matter more than avoiding an internet-facing route.

## Reliability model: separate trigger retries from recipient retries

Cron retries and email retries solve different failures. The former answers whether a report run exists; the latter answers whether each subscriber delivery reaches a terminal state. Mixing both into the request handler creates a nasty edge: subscriber 37 can receive an email, subscriber 38 can hit a rate limit, and the whole HTTP request can be retried. Without per-recipient keys, the first 37 deliveries may repeat.

Retries cross layers.

| Boundary | Useful guarantee | Persisted evidence | Recovery action |
| --- | --- | --- | --- |
| Cron to callback | A run is accepted at least once | Unique `run_key` | Retry the same signed payload |
| Outbox to worker | Each subscriber job remains recoverable | State, lease, and attempt count | Reclaim an expired lease |
| Worker to email provider | A send reaches a terminal result | Provider receipt or failure category | Back off or move the job to `dead` |

Model each outbox record as `pending`, `leased`, `sent`, or `dead`, with an attempt count and the next eligible time. A worker claims a bounded batch using a short lease, sends with an idempotency key if the email provider accepts one, and commits the result. On HTTP `429 Too Many Requests`, honor `Retry-After` when it is present and otherwise apply bounded exponential backoff with jitter. MDN documents both the meaning of `429` and the optional `Retry-After` response header. Don't spin.

Exactly-once email delivery is not a promise this boundary can prove by itself. If a worker sends successfully and stops before committing `sent`, the next worker may retry. Provider-side idempotency can narrow that gap; absent that capability, the honest contract is at-least-once processing with duplicate suppression where available. The report should include its business date clearly so a rare duplicate is recognizable, and the system should measure duplicates rather than burying the possibility. A dead-letter path then prevents a poisoned address or malformed personalization record from consuming retries forever. AWS's SQS documentation describes a dead-letter queue as a place for messages that were not processed successfully, controlled by a redrive policy and a maximum receive count. The general design applies even when the queue is a database table: preserve the failed job, last error category, attempt count, and next operator action. Do not automatically replay dead jobs until the underlying cause has been classified, because replay without a changed condition just repeats load. This is where prompt-cost awareness also matters in an AI-generated report. Generate shared summaries once per shipment cohort when the content permits it, record the model and prompt version beside the artifact, and fan out the stored artifact rather than regenerating it during every delivery retry. The evaluation harness should run before deployment against fixed shipment fixtures: missing scans, duplicated tracking events, late arrivals, and an empty day. Scheduling tests prove timing; evals prove that the report still says the right thing. Those are separate gates.

Delivery is not.

## Rollout: shadow the ledger before switching the timer

A useful dashboard starts with one row per business run: scheduled time, accepted time, subscriber count, pending count, sent count, dead count, and oldest pending age. Alert on a missing run shortly after its acceptance deadline and on an outbox whose oldest pending item exceeds the delivery objective. Raw worker error volume is secondary; ten retries that recover may matter less than one daily run that never existed. Correlate every log and metric with `run_key` and `subscriber_id`, while keeping email addresses out of routine logs.

Before deployment, exercise four transitions with a fixed clock: the first trigger creates a run, the identical trigger is a no-op, a `429` schedules a later attempt, and an exhausted attempt budget moves one recipient to the dead-letter path without blocking others. Then test a worker losing its lease after send, because that reveals the true duplicate boundary. Deploy the callback and worker independently, rotate the webhook secret without downtime by accepting old and new keys briefly, cap concurrency to protect the email provider, and rehearse a single-run replay in a non-production mailbox.

## Cost control: render once and evaluate before fan-out

For the notebook-to-production handoff, retain a tiny fixture with three subscribers and five shipment events. The notebook may explore wording or exception thresholds, but production owns deterministic interval selection, versioned prompts, persisted artifacts, and delivery state. This separation makes cost regressions and content regressions visible before the daily clock turns them into a subscriber-wide event.

Rendering once per cohort also puts a hard boundary around model calls: delivery retries read an existing artifact instead of spending tokens again. The catch is personalization. If every subscriber needs a materially different report, cohort rendering is not suitable; keep per-recipient generation, persist its result before attempting email, and include that path in the cost evaluation.

The final selection rule is plain: choose a cron service by its ability to make a signed, observable, retryable request with a stable payload, but judge the complete system by per-subscriber recovery. The scheduler starts the run. It cannot supply the delivery guarantee on its own.

## References

- AWS, "Using dead-letter queues in Amazon SQS": https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- MDN, "429 Too Many Requests": https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
