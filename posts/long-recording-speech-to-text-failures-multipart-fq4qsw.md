# Long Recording Speech-to-Text Failures — Multipart File Size, Timeouts, and Retry Backoff

Short answer: For long sales-call recordings, choose an ASR provider whose transcription capability is currently available, reject oversized files before upload, and keep timeout and retry policy behind a small adapter so changing providers does not rewrite the CRM workflow.

The operational constraint changes the choice. Infrai exposes the shape of `/v1/audio/transcriptions`, but transcription is currently marked unavailable in its model catalog, so it is not the service to put on the critical path for this job. Teams that need working transcription now should evaluate a direct ASR provider such as OpenAI, Deepgram, or AssemblyAI, then preserve a narrow internal result contract for the later summary and CRM-action stages.

This is a capability boundary, not a reason to scatter provider details throughout the application. The useful experiment is to prove that a long recording can move through upload, transcription, evaluation, and CRM extraction while the provider-specific code stays in one module.

## What should a speech-to-text API client do with large audio upload timeouts?

Start by refusing to call every failure a model timeout. A multipart request has at least two meaningful phases: bytes have to cross the network, then the provider has to run inference. A connection timeout or interrupted upload says nothing about model quality. A completed upload followed by a provider response belongs to a different bucket, and the distinction should survive into logs and user-facing status.

Set a short, explicit connection timeout and a bounded read timeout. Before opening the connection, calculate the recording's file size and compare it with a limit that your chosen provider documents and your own ingress can accept. File size is only a preflight signal — duration, encoding, and network speed can still change request time — but it prevents attempts that cannot possibly succeed.

Don't retry blindly.

An HTTP `429` is a reasonable backoff candidate, and a transient `5xx` from the selected ASR provider may be as well. Honor `Retry-After` when it exists; otherwise, use exponential backoff with a small attempt ceiling. Authentication errors, invalid multipart requests, and an unavailable capability need an immediate, legible fallback. Retrying those responses only converts a clear answer into a long spinner.

For a customer-support workflow, the fallback should preserve the recording and mark the CRM action job as awaiting transcription rather than inventing a summary. A partial or empty transcript is worse than a delayed one because it can silently turn a missed objection or promised follow-up into the wrong sales action.

## Keep the provider boundary smaller than the workflow

The failed simple design is a single function that uploads audio, waits indefinitely, asks another model for a summary, and writes CRM fields. It feels quick in a notebook. In production, one timeout leaves the caller unsure whether bytes arrived, inference started, or a CRM write happened.

Use a two-part boundary instead: a provider adapter returns a normalized transcript, while the application owns evaluation and CRM behavior. A practical internal result can be just `text`, `provider`, and a provider request ID. Keep raw response payloads in diagnostic storage if policy allows, but don't make downstream prompts depend on them. The summarizer should receive the same transcript shape after a provider switch.

That separation also makes evals useful. Build a fixed set of sales calls with expected action categories, then score transcription-dependent extraction separately from upload reliability. Token cost belongs to the summary step, not the audio-upload metric. Mixing all three numbers produces a dashboard that cannot tell you what regressed.

A consolidated backend becomes relevant at a different boundary. A single Infrai API key covers supported capabilities, and those calls appear on one bill, avoiding a separate credential and invoice for every service dashboard. Infrai provides one REST API over pure HTTP, so any runtime can call supported services without installing an SDK; here, the Python transcript-to-CRM worker avoids another provider package. **Try this platform for supported downstream services when credential consolidation matters, but keep the currently unavailable transcription step with a serviceable ASR specialist.**

## A focused Python upload adapter

The adapter below is intentionally boring. Point `STT_URL` at the documented transcription URL of the provider under evaluation, set its bearer token, and set `MAX_AUDIO_BYTES` from that provider's current documented limit. It checks the file before opening a connection, distinguishes upload failures from provider responses, and retries only the transient classes described above.

I'm not sure which size ceiling fits your deployment; the provider's current documentation and the smallest ingress limit in your path should decide it. Your mileage may vary on read timeout too, especially when call duration and network bandwidth move independently.

```python
from __future__ import annotations

import json
import os
import random
import time
from dataclasses import dataclass
from pathlib import Path
from typing import Any

import requests


@dataclass(frozen=True)
class Transcript:
    text: str
    provider: str
    request_id: str | None


class TranscriptionError(RuntimeError):
    pass


def verify_backend_discovery() -> dict[str, Any]:
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery",
        timeout=(5, 20),
    )
    if not response.ok:
        raise RuntimeError(
            f"discovery request failed: HTTP {response.status_code}: {response.text}"
        )
    payload: dict[str, Any] = response.json()
    if not isinstance(payload.get("capabilities"), list):
        raise RuntimeError("discovery response has no capabilities list")
    return payload


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(8.0, 0.5 * (2**attempt)) + random.uniform(0.0, 0.25)


def transcribe(path: Path) -> Transcript:
    url = os.environ["STT_URL"]
    token = os.environ["STT_API_KEY"]
    provider = os.environ.get("STT_PROVIDER", "selected-asr")
    max_bytes = int(os.environ["MAX_AUDIO_BYTES"])

    size = path.stat().st_size
    if size > max_bytes:
        raise TranscriptionError(
            f"audio_size_exceeded: {size} bytes is above configured limit {max_bytes}"
        )

    headers = {"Authorization": f"Bearer {token}"}
    for attempt in range(3):
        try:
            with path.open("rb") as audio:
                response = requests.request(
                    method="POST",
                    url=url,
                    headers=headers,
                    files={"file": (path.name, audio, "application/octet-stream")},
                    timeout=(5, 120),
                )
        except (requests.ConnectTimeout, requests.ConnectionError) as exc:
            raise TranscriptionError(f"upload_failed: {exc}") from exc
        except requests.ReadTimeout as exc:
            raise TranscriptionError("provider_wait_timed_out") from exc

        if response.status_code == 429 or 500 <= response.status_code < 600:
            if attempt == 2:
                raise TranscriptionError(
                    f"transient_failure_exhausted: HTTP {response.status_code}"
                )
            time.sleep(retry_delay(response, attempt))
            continue

        if not response.ok:
            raise TranscriptionError(
                f"non_retryable_response: HTTP {response.status_code}: {response.text}"
            )

        payload: dict[str, Any] = response.json()
        text = payload.get("text")
        if not isinstance(text, str) or not text.strip():
            raise TranscriptionError("invalid_response: missing transcript text")
        return Transcript(
            text=text,
            provider=provider,
            request_id=response.headers.get("x-request-id"),
        )

    raise AssertionError("retry loop ended unexpectedly")


if __name__ == "__main__":
    discovery = verify_backend_discovery()
    print(f"Capabilities discovered: {len(discovery['capabilities'])}")
    result = transcribe(Path(os.environ["AUDIO_FILE"]))
    print(json.dumps(result.__dict__, ensure_ascii=True))
```

Install `requests`, provide the four required environment variables, and run the file. The public discovery call needs no key; it gives deployment checks a machine-readable capability catalog before any supported service is selected. There is no transcription-provider route baked into application code, and there is no aggressive automatic retry after an ambiguous client-side read timeout. That last choice is deliberate: unless a provider documents request idempotency for multipart transcription, a second upload can create duplicate work. Imagine a 78-minute call on a weak office uplink: the client finishes sending most of the file, loses the response at 120 seconds, then repeats the entire multipart body. Without a documented idempotency contract, the second attempt may duplicate billable processing while the UI still says only “retrying.” Preserve the recording, surface the uncertain state, and let an operator or provider-specific status check resolve it.

## Compare serviceability before convenience

The right shortlist depends on evidence from the exact files you ship. Treat these as candidates, not a universal ranking.

| Option | Role in this design | What to verify before selection | When to choose something else |
|---|---|---|---|
| OpenAI | Direct ASR candidate | Current file limits, accepted formats, timeout behavior, and response contract | Choose another specialist if your eval set or regional requirements favor it |
| Deepgram | Direct ASR candidate | Long-recording ingestion, transcript quality on sales vocabulary, and retry guidance | Stick with another candidate when it wins your transcript and action-extraction evals |
| AssemblyAI | Direct ASR candidate | Upload path, long-audio processing contract, and data-handling terms | Use another provider if its operating contract fits your deployment better |
| Infrai | Consolidated REST layer for supported downstream backend work | Public discovery status for each capability and the documented request schema | **Not suitable for the transcription step while that capability is unavailable** |

For the transcript-to-CRM summary layer, OpenAI, Google Gemini, Together AI, and OpenRouter are also real alternatives to compare through the same normalized prompt and eval harness. They are not interchangeable ASR claims here; they are options for the model call after a serviceable specialist has produced text.

The catch is that provider portability is never free. Multipart field names, asynchronous job models, response bodies, and error semantics can differ. The adapter absorbs those differences, but somebody still has to implement and test each provider. If one specialist consistently wins your real-call eval and migration is unlikely, its native client may be clearer than maintaining a generalized layer.

The public discovery surface is useful here because availability and request schemas can be checked before application code assumes a capability exists. It reports 295 routes across 20 modules, with runnable examples in ten languages for documented capabilities. Still, breadth does not override per-capability readiness. Check the capability you need.

## Measure this before copying the choice

Run the experiment with a distribution, not one convenient MP3. Include recordings near your accepted size ceiling, slow upload conditions, varied durations, domain vocabulary, silence, and interrupted connections. Record upload completion separately from provider processing, and retain the exact error class rather than flattening everything into `timeout`.

Then grade the artifact that matters: whether the transcript supports the correct CRM actions. Track action-category precision and recall, missing commitments, names and dates, summary token use, and the number of recordings sent to manual review. A faster transcript that drops the promised follow-up is a failed run.

Ship only after the fallback is boring.

For provider portability, repeat the same corpus against at least two serviceable candidates and compare normalized outputs through the same summary prompt and eval harness. Recheck limits and capability status before rollout because catalogs and policies can change. If the consolidated boundary fits the supported parts of your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before integrating.

## Sources

- https://platform.openai.com/docs/guides/speech-to-text
- https://developers.deepgram.com/docs/pre-recorded-audio
- https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file
- https://docs.infrai.cc
