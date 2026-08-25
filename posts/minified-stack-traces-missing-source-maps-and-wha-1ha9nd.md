# Minified stack traces, missing source maps, and what error tracking can't fix

A minified stack trace in production means the bundle the browser executed is not the source anyone wrote, and the map that reverses that transform was either never built, never uploaded, or never reachable by whatever tries to symbolicate it. So fix the artifact pipeline first, and use the error tracker for grouping, alerting and rate limiting rather than for archaeology — it can only resolve frames it holds a map for.

Everything below is downstream of that one sentence.

The system I keep coming back to is a nightly ingest pipeline in a healthtech product. Python jobs normalize lab results in the small hours, a Next.js console lets clinical ops review whatever failed, and a React worker in a tab that ops leaves open all night drives the retry queue. Assume tens of thousands of structured log events per run, of which maybe a dozen describe a bug worth paging a human for. Signal quality versus noise is the entire design constraint, and a stack trace that says `a.b is not a function` at `chunk-4f2c.js:1:88214` is pure noise: it can't be grouped, can't be assigned, can't be regression-tested.

## Why do React and Next.js errors reach production with minified stack traces?

Because `Error.prototype.stack` is assembled from the frames that are actually running. The bundler renamed the identifiers, inlined the modules and dropped the line breaks, so the runtime honestly reports what it executed. Nothing is broken at that point — the information needed to translate it back simply lives in a separate file.

Two mechanisms carry that file: a `//# sourceMappingURL=` comment at the end of the emitted chunk, or a `SourceMap` HTTP response header on the chunk itself. Both are part of the source map format specification, and both are trivially easy to lose. Browser source maps for production builds are opt-in in Next.js (`productionBrowserSourceMaps`), and teams switch them off deliberately, since a publicly fetchable map hands out readable source to anyone who opens the network tab. Turning them off is a defensible security call. Forgetting that the tracker then has nothing to work with is the part that hurts.

There's a second truncation nobody expects. V8 captures a limited number of frames — `Error.stackTraceLimit` defaults to 10 — so in a deep React render path or a promise chain the frame you actually need may never be captured, map or no map. Errors from a cross-origin script arrive as a bare `Script error.` unless the tag carries `crossorigin` and the CDN returns the matching CORS header. In both cases the source map is a red herring; the trace was already lossy before it left the browser.

## What actually goes missing, and where

| Failure mode | What you see | Where it gets fixed |
| --- | --- | --- |
| Map never emitted | Frames point at chunk files, every release | Build config |
| Map emitted, never uploaded | Frames resolve locally, not in the tracker | Release/publish step |
| Map uploaded under a different build id | Frames resolve to the wrong lines | Version stamping |
| Map served, then deleted from the CDN | Old releases stop resolving | Artifact retention |
| Trace truncated before the app frame | Shallow stack, all framework frames | `Error.stackTraceLimit` |

Only the first row is a build problem. The other four are release-engineering problems, which is why "we enabled source maps" so often fails to change anything: the map has to be produced, stamped with the same version string the runtime reports, stored somewhere durable, and kept for as long as an old bundle can still be sitting in someone's open tab. Ops tabs stay open for days.

## The grep-it-later approach, and the one that replaced it

The naive setup is to ship everything to a log store and search it in the morning. It works for about a week. Then a single retry storm writes tens of thousands of near-identical lines, full-text search over that night returns a wall of results, and the one genuinely new exception is buried on page nine. Log volume grows with traffic; the number of distinct bugs doesn't.

What survives contact with a nightly batch is fingerprinting at write time — collapse each event into the tuple a human would call "the same bug", then rank by novelty rather than by count. This one runs as a Python job right after the window closes, over events shaped by the OpenTelemetry logs data model, so `severity_number`, `service.version` and the `exception.*` attributes are already there.

```python
"""Nightly triage: one run of structured logs -> a ranked list of distinct bugs."""
import json
import re
from collections import Counter
from pathlib import Path

# A frame still pointing at a bundled chunk means symbolication did not happen.
MINIFIED_FRAME = re.compile(r"/chunks/[\w.-]+\.js:\d+:\d+|\.min\.js:\d+:\d+")
DROP_ATTRS = {"patient.mrn", "patient.name", "patient.dob"}  # never leaves the cluster

ERROR = 17  # severity_number for ERROR in the OpenTelemetry logs data model


def scrub(event: dict) -> dict:
    return {k: v for k, v in event.items() if k not in DROP_ATTRS}


def fingerprint(event: dict) -> tuple:
    frames = [f.strip() for f in event.get("exception.stacktrace", "").splitlines() if " at " in f]
    return (
        event.get("exception.type", "Error"),
        frames[0] if frames else "no-frame",
        event.get("service.version", "unknown"),
        event.get("feature.retry_v2", "off"),  # toggle state: same class, different bug
    )


def triage(path: Path):
    groups, resolved, total = Counter(), 0, 0
    for line in path.read_text(encoding="utf-8").splitlines():
        event = scrub(json.loads(line))
        if event.get("severity_number", 0) < ERROR:
            continue
        total += 1
        stack = event.get("exception.stacktrace", "")
        if stack and not MINIFIED_FRAME.search(stack):
            resolved += 1
        groups[fingerprint(event)] += 1
    return groups, (resolved / total if total else 0.0), total


groups, symbolication_rate, total = triage(Path("logs/nightly.ndjson"))
print(f"{total} error events, {len(groups)} distinct, {symbolication_rate:.0%} symbolicated")
for key, count in groups.most_common(10):
    print(count, key)
```

The toggle state in the fingerprint is doing real work. If a risky pipeline step sits behind a release toggle, the same `TypeError` with the toggle on and with it off are two different bugs with two different owners, and Martin Fowler's taxonomy of toggle categories is a good sanity check on which ones deserve a fingerprint field at all — release toggles do, permission toggles rarely do.

Then there's the artifact gate, which is the cheaper half and the one I'd write first:

```python
from pathlib import Path

MARKER = "//# sourceMappingURL="


def missing_maps(build_dir: Path) -> list[str]:
    gaps = []
    for js in build_dir.rglob("*.js"):
        text = js.read_text(encoding="utf-8", errors="ignore")
        if MARKER not in text:
            gaps.append(f"{js}: no sourceMappingURL")
            continue
        ref = text.rsplit(MARKER, 1)[1].splitlines()[0].strip()
        if not ref.startswith("data:") and not (js.parent / ref).exists():
            gaps.append(f"{js}: map {ref} was never produced")
    return gaps
```

Run it as a build step, fail the release on a non-empty list, then upload the maps to private storage and delete them from the public output directory. Two dozen lines, and it removes an entire class of "why is this trace minified" mornings.

## Where error tracking stops helping

Hosted error trackers are very good at one shape of problem: an exception was thrown, in a browser, with a version string attached. The limitations show up the moment the failure isn't an exception. A nightly pipeline that quietly writes a fifth of the rows it should threw nothing at all — no stack, minified or otherwise. That's a data-quality assertion, and it belongs in the pipeline's own checks, not in an error tracker.

The catch is retention and sampling. Client-side error volume is dominated by a handful of noisy browser extensions and network blips, so every tracker samples or rate-limits somewhere, and the event you most want is by definition rare. If your triage question is "did this specific bug happen last Tuesday at 02:14", stick with searchable structured logs and accept the storage bill. Tracking tools answer "what is breaking most", which is a different question.

The other trade-off is regulatory. Under HIPAA, a stack trace containing a request payload can carry PHI straight out of your boundary, so healthtech teams redact aggressively before egress — and aggressive redaction removes exactly the context that made grouping precise. I don't have a clean answer for that tension. Redact at the collector, keep the identifiers as salted hashes so events still join, and expect to lose some grouping quality.

## What to measure before you copy this

Symbolication rate is the first number: the share of error events with at least one resolved application frame. If it isn't near-total after a release, your maps and your version stamps disagree, and no amount of tracker configuration will fix it.

After that, count distinct fingerprints per night rather than events per night, and watch how many pages led to a code change within a week. That ratio is the honest measure of signal quality; event volume just measures traffic. Your mileage may vary on the threshold, but a night that produces more unique fingerprints than a human can read in ten minutes is a triage design problem, not an alerting problem.

**Ship the map with the release, stamp both with the same version, and fingerprint before you search.** The rest is tooling preference.

## Further reading

- Source map format specification (TC39): https://tc39.es/source-map/
- MDN, "Source map": https://developer.mozilla.org/en-US/docs/Glossary/Source_map
- V8 stack trace API and `Error.stackTraceLimit`: https://v8.dev/docs/stack-trace-api
- Node.js CLI: `--enable-source-maps`: https://nodejs.org/api/cli.html#--enable-source-maps
- Next.js config: `productionBrowserSourceMaps`: https://nextjs.org/docs/app/api-reference/config/next-config-js/productionBrowserSourceMaps
- OpenTelemetry logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- OpenTelemetry semantic conventions for exceptions: https://opentelemetry.io/docs/specs/semconv/exceptions/exception-logs/
- Martin Fowler, "Feature Toggles": https://martinfowler.com/articles/feature-toggles.html
- HHS, HIPAA Security Rule: https://www.hhs.gov/hipaa/for-professionals/security/index.html
