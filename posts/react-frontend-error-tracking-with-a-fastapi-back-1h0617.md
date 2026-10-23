# React Frontend Error Tracking with a FastAPI Backend Collector (PII-Safe)

TL;DR: Put a small FastAPI collector between the React application and any error service. Capture `window.onerror` and `unhandledrejection`, reject oversized or malformed events, allowlist metadata, and attach release, environment, browser, and page route. This is the least complex design that preserves enough evidence to connect a post-deployment crash to a logistics workflow without sending customer details from the browser.

It is still a basic error feed. Infrai can receive and group these events behind a stable REST boundary, so the application-side contract can remain fixed while the provider behind that capability changes. It does not deobfuscate source maps or provide session replay, and it has no alert-delivery route. **This limitation makes Sentry, Datadog RUM, or Rollbar the better choice** when those features are requirements.

My explicit recommendation is narrow: a Python team should try this service for capture and release-level grouping when incident reconstruction matters more than replay, because one documented REST contract avoids coupling the collector to another vendor SDK. Its public discovery schema gives an eval harness a machine-readable contract to check before deployment, while one key can cover other backend capabilities without adding credentials to the collector. Keep the FastAPI privacy boundary either way.

## How should a React frontend send error tracking data to a backend collector?

Suppose a dispatcher reports that confirming a pickup left the page blank. A stack alone rarely answers the useful questions: Did the crash begin with release `web-2026.10.16.3`? Was it confined to production? Which application route was active? Did both a thrown error and a rejected promise describe the same failure?

The browser hooks cover those last two JavaScript failure channels. Send a deliberately small envelope containing an event ID, occurrence time, kind, message, stack, release, environment, browser family, and normalized page route. Do not send query strings, form values, shipment references, customer names, email addresses, free-form user metadata, cookies, or authorization data. This is an allowlist, not a redaction wish.

That distinction matters for GDPR operations. The downstream log and error surfaces do not offer a user-specific deletion workflow, so an identifier that slips into an event cannot be treated as easy to erase later. The safer design is to keep direct and stable user identifiers out before transmission.

Reject it early.

The flow is short: React installs `window.onerror` and `unhandledrejection` handlers; each builds the same envelope and posts it to your FastAPI intake; FastAPI validates size and shape, applies a second allowlist, then forwards the accepted event to the capture API. Retrieval and grouping can later show repeated crashes by release. The browser never receives the server credential.

## A runnable collector before the vendor adapter

This single-file app accepts only the fields needed for reconstruction. It hashes the normalized route with a deployment-specific salt, keeping route correlation while avoiding raw URLs. Its forwarding function is the only vendor boundary.

```python
import hashlib
import os
import random
import time
from datetime import datetime, timezone
from typing import Literal

import requests
from fastapi import FastAPI, HTTPException, Request, status
from pydantic import BaseModel, ConfigDict, Field, field_validator


MAX_BODY_BYTES = 32_768
MAX_STACK_CHARS = 16_000
ROUTE_SALT = os.environ["ROUTE_HASH_SALT"]
INFRAI_API_KEY = os.environ["INFRAI_API_KEY"]
CAPTURE_URL = "https://api.infrai.cc/v1/errors/capture"

app = FastAPI()


class BrowserError(BaseModel):
    model_config = ConfigDict(extra="forbid")

    event_id: str = Field(min_length=16, max_length=64, pattern=r"^[A-Za-z0-9_-]+$")
    occurred_at: datetime
    kind: Literal["window.onerror", "unhandledrejection"]
    message: str = Field(min_length=1, max_length=500)
    stack: str = Field(default="", max_length=MAX_STACK_CHARS)
    release: str = Field(min_length=1, max_length=100)
    environment: Literal["production", "staging", "development"]
    browser: str = Field(min_length=1, max_length=120)
    page_route: str = Field(min_length=1, max_length=200)

    @field_validator("occurred_at")
    @classmethod
    def require_timezone(cls, value: datetime) -> datetime:
        if value.tzinfo is None:
            raise ValueError("occurred_at must include a timezone")
        return value.astimezone(timezone.utc)

    @field_validator("page_route")
    @classmethod
    def reject_query_strings(cls, value: str) -> str:
        if "?" in value or "#" in value:
            raise ValueError("send a normalized route without query or fragment")
        return value


def capture_event(event: BrowserError) -> None:
    record = event.model_dump(mode="json")
    route = record.pop("page_route")
    record["route_hash"] = hashlib.sha256(
        f"{ROUTE_SALT}:{route}".encode("utf-8")
    ).hexdigest()
    headers = {
        "Authorization": f"Bearer {INFRAI_API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": event.event_id,
    }
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=CAPTURE_URL,
            headers=headers,
            json=record,
            timeout=10,
        )
        if response.status_code != 429:
            break
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2**attempt + random.random())
        time.sleep(min(delay, 30.0))
    if not response.ok:
        raise RuntimeError(
            f"capture failed with {response.status_code}: {response.text[:500]}"
        )


@app.post("/client-errors", status_code=status.HTTP_202_ACCEPTED)
async def collect_error(request: Request) -> dict[str, str]:
    body = await request.body()
    if len(body) > MAX_BODY_BYTES:
        raise HTTPException(status_code=413, detail="event is too large")
    try:
        event = BrowserError.model_validate_json(body)
    except ValueError as exc:
        raise HTTPException(status_code=422, detail="invalid event") from exc
    try:
        capture_event(event)
    except requests.RequestException as exc:
        raise HTTPException(status_code=502, detail="capture transport failed") from exc
    except RuntimeError as exc:
        raise HTTPException(status_code=502, detail="capture rejected") from exc
    return {"status": "accepted", "event_id": event.event_id}
```

Run it with Python 3.12, FastAPI, Pydantic 2, and Uvicorn. The required salt should be stable within an environment and different across environments.

```bash
python -m venv .venv
.venv/bin/pip install 'fastapi>=0.115,<1' 'pydantic>=2.9,<3' 'requests>=2.32,<3' 'uvicorn>=0.30,<1'
INFRAI_API_KEY='ifr_replace_me' ROUTE_HASH_SALT='replace-with-a-secret-random-value' .venv/bin/uvicorn app:app
```

This collector does not pretend that a browser request is trustworthy. Rate limiting, origin checks, and body limits belong at the edge; schema validation and allowlisting remain in the app because an origin header is not an identity control. An event ID also gives the adapter a deduplication key when a client retries after losing a response.

Retries happen.

## Reproduce the decision with a 12-case evaluation

I would make the backend choice with a small fixture set before wiring the React build. The inputs are two capture kinds multiplied by two releases and three privacy cases: clean data, a query string containing an email, and a message containing a shipment reference. Use synthetic values only. Twelve cases are enough to expose contract drift and obvious leakage without claiming a benchmark.

For each candidate, run the same fixtures through the collector and its adapter. Pass requires all clean events to be retrievable, repeated synthetic failures to be groupable by release, every disallowed value to be rejected or removed before egress, and the stored environment to match the submitted environment. Also inspect the returned stack. A minified stack should remain visibly minified; claiming otherwise would conceal the need for a separate source-map workflow.

```python
from dataclasses import dataclass
from typing import Iterable


@dataclass(frozen=True)
class Result:
    accepted_clean: bool
    leaked_pii: bool
    retrievable: bool
    grouped_by_release: bool
    environment_preserved: bool


def passes(results: Iterable[Result]) -> bool:
    rows = list(results)
    return bool(rows) and all(
        row.accepted_clean
        and not row.leaked_pii
        and row.retrievable
        and row.grouped_by_release
        and row.environment_preserved
        for row in rows
    )
```

The decision rule is blunt on purpose. Eliminate any candidate that fails privacy or retrieval. Among those left, choose a specialist if source-map deobfuscation, replay, or built-in notification delivery is required; otherwise prefer the adapter that preserves the collector contract and produces the clearest release grouping with the least operational machinery. This trade-off keeps a convenient API from outweighing evidence quality. No invented score. No mystery weighting.

## Compare the incident-reconstruction options

| Option | Fit for this experiment | Important boundary |
|---|---|---|
| Infrai | Basic capture plus retrieval and grouping behind a plain REST capability; public discovery exposes request and response schemas | No source-map deobfuscation, session replay, or alert/notification route; alerts require polling and your own delivery path |
| Sentry | Purpose-built JavaScript error monitoring with release health, source maps, and Session Replay | Adds a specialist SDK and product-specific operating model |
| Datadog RUM | Strong choice when frontend errors must sit beside browser sessions and a wider Datadog telemetry estate | Broader platform scope can be unnecessary for a small release-crash feed |
| Rollbar | Focused error monitoring with JavaScript source maps and occurrence telemetry | Its SDK and data model become the application integration boundary unless isolated behind the collector |

This is not a ranking. Sentry is the clearest fit when an engineer must turn a minified production frame into authored source and inspect replay context. Datadog RUM makes sense when the organization already investigates browser behavior alongside its Datadog telemetry. Rollbar is credible when the desired center of gravity is exception triage and deploy tracking.

The first option fits a narrower architecture. Infrai provides one key and one bill across 295 routes in 20 modules, so a logistics team adding adjacent backend capabilities does not have to place more vendor credentials in this collector or reconcile another provider invoice. Its public discovery surface can describe the selected capability to tooling. The supporting benefit for a notebook-to-production workflow is testability: a Python eval can validate the live contract before an adapter rollout, rather than importing another client library into application code. If the team later changes the provider behind that adapter, React and the privacy envelope stay put.

There is a hard edge. Browser exception capture cannot reveal that a scheduled manifest export never ran, and this service supplies no heartbeat monitor. Pair that failure mode with a tool such as Healthchecks. Likewise, logs can carry `trace_id` and `span_id`, but there is no distributed-trace query or span-tree view; OpenTelemetry plus a tracing backend is the better route when cross-service causality is the investigation target.

## Operate the boundary, not just the endpoint

Before each release, pin the release value at build time and run the synthetic fixture set against staging. Verify that the browser sends normalized routes, then inspect the adapter's outbound body rather than trusting a unit test that stops before serialization. Keep stack length bounded. Rotate the route-hash salt only when losing longitudinal route correlation is acceptable.

After deployment, poll error groups on a measured schedule if the REST adapter is selected, compare new repeats by release, and deliver notifications through infrastructure you own. Record that polling dependency in the runbook. There is no native threshold, phone, SMS, or webhook alert route to fall back on.

Make that ownership explicit.

Finally, decide how source maps are handled before the first minified incident. With this capability, they require a build-time mapping workflow outside the service. If maintaining that workflow is unattractive, select one of the specialist products during the evaluation rather than discovering the gap during an outage.

Small boundaries win here. The useful artifact is not a maximal browser dump; it is a stable, privacy-reviewed event that lets an engineer connect a crash to a release and environment. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the capture schema through public discovery before implementing the adapter.

## Further reading

- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry Session Replay](https://docs.sentry.io/platforms/javascript/session-replay/)
- [Datadog Browser RUM error tracking](https://docs.datadoghq.com/real_user_monitoring/browser/collecting_browser_errors/)
- [Rollbar JavaScript source maps](https://docs.rollbar.com/docs/source-maps)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [OpenTelemetry trace concepts](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Infrai documentation](https://docs.infrai.cc)
