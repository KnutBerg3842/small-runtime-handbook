# Implementing Avatar Uploads: Safely Handling Existing Files Amid Object Storage Races

Short answer: upload every replacement to a new object key, verify it, then atomically change the authenticated customer's database pointer only if the pointer version still matches the version the uploader read. Don't overwrite the currently visible object. This keeps an avatar replacement from mixing old and new state, and the same publication pattern lets an e-commerce system stream large generated reports without holding a database transaction open during transfer.

The evaluation constraint matters more than the storage brand: a reader must observe either the complete previous file or the complete new file, while two writers may finish in either order. The tempting implementation, `avatars/{user_id}.jpg`, makes the blob itself both the upload destination and the public identity. A retry, a concurrent browser tab, or delayed cleanup can then act on a name whose meaning has changed.

The chosen design separates bytes from meaning. An immutable key identifies one byte sequence; a small, versioned row says which sequence is current. That extra pointer update is the useful synchronization point.

## How should an avatar upload replace an existing file without an object storage race condition?

Treat replacement as publication, not mutation. After authentication and authorization, the server allocates a key such as `tenants/t-42/avatars/u-7/01J...`, streams the body to that unused location, validates the stored result, and attempts a compare-and-swap on the user's metadata row. If the row was at version 8, the update succeeds only while it is still at version 8. A competing request that already advanced it to version 9 makes the late update lose cleanly.

This ordering gives the operation a precise commit point: the database update. Before that update, the new object is staged and invisible to normal reads. After it, new reads resolve to the new key. The old key remains a valid immutable object until a background retention policy removes it. No reader has to infer whether an in-place overwrite has propagated, and no cleanup worker should delete “the old avatar” by a reusable name.

An HTTP precondition can reinforce the boundary. RFC 9110 defines `If-Match` for making a request conditional on the selected representation's entity tag; a failed precondition is reported as `412 Precondition Failed`. If an API exposes the avatar metadata version as an ETag, a client can send `If-Match` and learn that its edit was stale. Keep the server-side compare-and-swap anyway. Client preconditions improve the protocol, but the authoritative race still has to close at the metadata write.

Do not assume an object-store ETag is a portable content hash. Provider and upload-mode semantics differ. Store an application-computed SHA-256 digest when integrity or deduplication decisions need a stable content identity, and reserve the row version for concurrency. Those values answer different questions.

## A focused Python concurrency experiment

The following standard-library program models the publication boundary. The in-memory object map stands in for an object store, while SQLite supplies the conditional metadata update. Both workers read version 1. They upload different bytes under different keys. Exactly one can change the pointer; the other removes only its own unpublished object.

```python
from __future__ import annotations

import hashlib
import io
import sqlite3
import threading
import uuid
from dataclasses import dataclass
from typing import BinaryIO


class VersionConflict(Exception):
    pass


class MemoryObjectStore:
    def __init__(self) -> None:
        self._objects: dict[str, bytes] = {}
        self._lock = threading.Lock()

    def put_new(self, key: str, source: BinaryIO, chunk_size: int = 1024 * 1024) -> str:
        digest = hashlib.sha256()
        parts: list[bytes] = []
        while chunk := source.read(chunk_size):
            digest.update(chunk)
            parts.append(chunk)

        with self._lock:
            if key in self._objects:
                raise FileExistsError(key)
            self._objects[key] = b"".join(parts)
        return digest.hexdigest()

    def delete(self, key: str) -> None:
        with self._lock:
            self._objects.pop(key, None)


@dataclass(frozen=True)
class PublishedFile:
    key: str
    version: int
    sha256: str


def publish(
    db: sqlite3.Connection,
    store: MemoryObjectStore,
    tenant_id: str,
    customer_id: str,
    expected_version: int,
    source: BinaryIO,
) -> PublishedFile:
    object_id = uuid.uuid4().hex
    key = f"tenants/{tenant_id}/customers/{customer_id}/files/{object_id}"
    digest = store.put_new(key, source)

    cursor = db.execute(
        """
        UPDATE customer_files
           SET object_key = ?, sha256 = ?, version = version + 1
         WHERE tenant_id = ? AND customer_id = ? AND version = ?
        """,
        (key, digest, tenant_id, customer_id, expected_version),
    )
    db.commit()
    if cursor.rowcount != 1:
        store.delete(key)
        raise VersionConflict(f"expected version {expected_version}")

    return PublishedFile(key, expected_version + 1, digest)


def run_worker(
    db_path: str,
    store: MemoryObjectStore,
    label: str,
    barrier: threading.Barrier,
) -> None:
    connection = sqlite3.connect(db_path, timeout=5)
    barrier.wait()
    try:
        result = publish(
            connection,
            store,
            tenant_id="shop-42",
            customer_id="customer-7",
            expected_version=1,
            source=io.BytesIO(f"report generated by {label}".encode()),
        )
        print(label, "published", result.version, result.key)
    except VersionConflict as error:
        print(label, "conflict", error)
    finally:
        connection.close()


def main() -> None:
    db_path = "publication-demo.sqlite3"
    connection = sqlite3.connect(db_path)
    connection.execute("DROP TABLE IF EXISTS customer_files")
    connection.execute(
        """
        CREATE TABLE customer_files (
            tenant_id TEXT NOT NULL,
            customer_id TEXT NOT NULL,
            object_key TEXT NOT NULL,
            sha256 TEXT NOT NULL,
            version INTEGER NOT NULL,
            PRIMARY KEY (tenant_id, customer_id)
        )
        """
    )
    connection.execute(
        "INSERT INTO customer_files VALUES (?, ?, ?, ?, ?)",
        ("shop-42", "customer-7", "initial", "", 1),
    )
    connection.commit()
    connection.close()

    store = MemoryObjectStore()
    barrier = threading.Barrier(2)
    workers = [
        threading.Thread(target=run_worker, args=(db_path, store, label, barrier))
        for label in ("worker-a", "worker-b")
    ]
    for worker in workers:
        worker.start()
    for worker in workers:
        worker.join()


if __name__ == "__main__":
    main()
```

The winning label is intentionally nondeterministic. The invariant isn't. One line prints `published 2`; the other prints a conflict for expected version 1. Run this as an evaluation, not as production storage code: execute it repeatedly, assert one winner per starting version, and inspect that a losing key is absent. A production adapter should stream parts directly rather than collect them in a list, preserve the “create only if absent” condition offered by the selected storage service, and record the staged object before attempting publication so abandoned uploads can be found after a process exit.

There is one sharp edge in the example: `commit()` and object deletion do not share a transaction. No ordinary database can atomically commit a row and delete a remote blob. Production code should write an outbox or garbage-collection record in the same database transaction as the pointer change, then let an idempotent worker perform delayed deletion. Immediate deletion is safe only for the losing request's newly allocated, never-published key; even there, a periodic sweep is still needed for a process that stops between upload and cleanup.

## Why large generated reports change the evaluation

Avatar files make the race easy to reproduce, but generated e-commerce reports expose the throughput cost. A report may be read once, may be regenerated after a prompt or data revision, and can be large enough that proxy buffering, Python heap growth, and retry amplification dominate the metadata update. Keep the object transfer outside the database transaction. The transaction should contain a conditional row update and an outbox insertion, not minutes of network I/O.

For a notebook-to-production path, start with a deterministic fixture: the same orders, model output, report format, and authorization claims. Then test at least two sizes, including one above the threshold where the storage client switches to multipart or resumable transfer. Measure end-to-end publish latency, bytes buffered in the application process, retry bytes, time spent holding a database connection, stale-version conflict rate, and time until unreferenced objects are reclaimed. Prompt token cost belongs beside those measurements for an AI-generated report, because a concurrency retry should never silently rerun an expensive generation step. Persist the generated artifact identity before publication and retry only the transfer or pointer change.

Short files can hide a bad architecture.

For downloads, authorize the customer against the metadata row first, resolve the immutable key, and then stream or issue a short-lived delegated URL according to the threat model. Never accept a caller-provided object key as proof of authorization. Tenant and customer segments in the key help operations and lifecycle rules, but they are organization, not access control. The OWASP File Upload Cheat Sheet likewise recommends generated filenames, authorization, size limits, allowed-type checks, and storage outside the web root; content validation must happen before the pointer becomes visible.

I'm not sure what retention delay is right for every report workload. It depends on the longest valid download lease, cache behavior, audit requirements, and how quickly customers expect rollback. Resolve that uncertainty with access logs and an explicit recovery objective, then set a measured grace period. Seven days might be sensible for one audited report pipeline and wasteful for a replaceable avatar; it isn't a universal default.

## Failure modes worth putting in the eval harness

Make the race visible.

Test the state machine, not just the happy-path response. Pause worker A after it uploads a 640 MB report under key A, allow worker B to publish key B from the same starting version, and then release A. A must receive a conflict, while every new authorized read resolves to B. Next, begin a download of version 12, publish version 13, and invoke cleanup while the version 12 stream is active; the old bytes must remain available for the promised lease. Retry B after simulating a lost client response and verify that its scoped idempotency key returns the recorded result instead of storing another copy. Reject a stale metadata request with HTTP 412 at the API boundary, then confirm that the stored pointer did not move. Finally, attempt a cross-tenant key substitution and verify that authorization follows the row's tenant ownership rather than the path text. The useful assertions stay compact even though the schedule is messy: each committed version maps to exactly one immutable key; a key's bytes never change; a failed compare-and-swap never changes the visible pointer; and garbage collection deletes only keys that are unreferenced and older than the retention boundary. Add digest verification after upload when corruption detection is required. Record `tenant_id`, logical file ID, old and new versions, object size, digest, request ID, and publish outcome in structured logs, but don't log delegated download credentials. A client-supplied idempotency key must be scoped to the authenticated customer and operation, and it must not bypass the expected-version check for a genuinely different replacement. This is where a quick notebook often cheats: it treats “same request ID” and “same bytes” as the same concept. They aren't.

Order matters.

## When should you choose a different strategy?

Immutable keys plus a database pointer are a strong fit when reads greatly outnumber replacements, old bytes must remain readable during a transition, or large-file transfer time must be isolated from the commit. The catch is operational: the design needs metadata, lifecycle cleanup, observability, and a recovery procedure for staged objects. It also consumes additional storage during the retention window.

Stick with a conditional in-place overwrite when the object name is itself an external contract, the storage system provides a well-understood generation or version precondition, readers can tolerate that store's documented consistency and cache semantics, and retaining the prior key has no value. Use built-in object versioning when compliance requires recoverable generations and its retention model matches the policy. For collaborative documents with frequent partial edits, object replacement is the wrong primitive; use a transactional document system or an append-only event model instead.

Before copying this choice, measure the largest real report rather than a synthetic 1 MB avatar, confirm that the SDK can stream without whole-body buffering, and force the exact interleavings described above. Also price retained bytes, abandoned multipart uploads, database writes, and egress separately. Cost matters, but correctness and bounded memory are the first gates: an inexpensive path that republishes stale customer data is still a failed design.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [Python `hashlib` documentation](https://docs.python.org/3/library/hashlib.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
