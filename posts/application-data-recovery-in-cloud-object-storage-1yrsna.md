# Application Data Recovery in Cloud Object Storage: Signed URLs for Archives

## TL;DR

For application data backups, begin with the storage design that gives the restore copy an independent failure boundary, then make a verified restore script and a time-limited signed URL policy part of the design from day one. A self-managed object store can meet that bar, but putting it beside the application does not; cloud object storage can reduce the storage operation burden, while leaving retention, identity separation, cost modeling, and recovery evidence with the application team.

The deciding constraint is recoverability, not the cheapest-looking byte rate. A backup that has never been restored by a separate identity is an assertion, not evidence.

Evidence wins.

## What should app data backups, cloud object storage, restore scripts, and signed URLs prove?

This architecture decision record uses four invariants. First, the application credential that writes a backup should not also be able to erase every recoverable copy. Second, every archived artifact needs a manifest that identifies its format, byte count, digest, source, and creation time. Third, a clean machine with the documented recovery identity must be able to retrieve and validate an artifact. Fourth, the application-specific import must be tested after transport verification; a matching SHA-256 digest says that bytes arrived intact, not that a database dump represents a coherent snapshot.

Those invariants separate several failures that are often collapsed into the word "backup." A partial upload, a stale manifest, an over-broad credential, a retention rule that removes the object before its manifest, and an importer that cannot consume the chosen dump are different faults with different owners. Treating them as one concern makes incident diagnosis slow and produces alarming confidence before an actual restore.

Name the fault.

The common beginner mistake is structural. Running an object service on the same machine, filesystem, administrator account, power path, and network boundary as the application creates a second process, not an independent recovery boundary. The service may expose an object API correctly and still disappear in the same loss event as the data it was meant to protect. A separate disk is useful, but it does not by itself answer the questions about host loss, credential compromise, or operator error.

| Decision axis | Self-managed object storage | Cloud object storage |
| --- | --- | --- |
| Primary operational owner | The application team owns media, upgrades, monitoring, replacement, and service recovery. | The provider operates the storage service; the application team owns object policy, identities, retention, and restore correctness. |
| Failure-boundary test | The storage nodes, credentials, and copies must survive loss of the application environment. | The application must still separate backup authority from production authority and test its policy assumptions. |
| Cost questions | Include hardware replacement, independent replication, observability, certificates, and operator time. | Include stored bytes, request classes, retrieval, and data transfer where the selected service charges for them. |
| Appropriate use | A team already operating independent infrastructure can make the boundary explicit. | A team that lacks that operational capacity can avoid adding a storage service to its first recovery path. |

This is deliberately not a product ranking. Both columns can be sound, and both can fail badly. The useful comparison is who can demonstrate the invariants, how failure boundaries are enforced, and how quickly a new operator can perform recovery under pressure.

The bucket label proves nothing.

## The recovery boundary is a manifest, not a bucket name

Store backup data as immutable artifacts described by a small manifest. The manifest should travel through version control or a controlled catalog with the same retention intent as the objects it names. It can include an object key, a byte count, a digest, an artifact format version, and the source snapshot identifier. Keep it boring. A restore operator should be able to inspect one document and know what is expected before downloading anything.

There is a subtle boundary here. Object storage provides a place to retain bytes and object metadata; it does not define whether the application created a transactionally consistent database export. The producer has to use the database's documented snapshot or export procedure, and the recovery exercise has to invoke the real importer. Copying a live database file can look successful while capturing an unusable point in time. Don't let a transport success code become the acceptance criterion.

Signed URLs belong to a different layer. They delegate narrowly scoped, time-limited access to a particular object operation, which can let a client transfer a large object without relaying its bytes through the application server. The authorization decision still happens before the URL is issued. Bind the intended HTTP method and object key, keep the expiry proportional to the transfer, avoid recording the full query string in logs, and validate the completed artifact on the trusted side. A signed URL is not a retention policy, a checksum, or a restore plan.

Authority expires. Recovery evidence should not.

For browser transfers, visible progress is also separate from successful archival. MDN documents upload progress event listeners on the upload target and warns that registering them can affect whether a cross-origin request remains a "simple request." Test the actual browser and CORS policy with the listener installed. A progress bar reports movement of bytes; it cannot attest to the manifest, the final object, or the future importer.

## A restore script should fail before importing data

The critical path is intentionally small: parse a manifest, reject unsafe names, stream the object into a temporary file, verify its declared length and SHA-256 value, then atomically expose it to the application-specific import command. The generic reader below keeps the transport choice outside the validation logic. A client for a cloud bucket, a self-managed endpoint, or a test double can implement the same narrow interface.

```python
from __future__ import annotations

import hashlib
import json
import os
from dataclasses import dataclass
from pathlib import Path, PurePosixPath
from typing import BinaryIO, Protocol


class ObjectReader(Protocol):
    def open(self, key: str) -> BinaryIO: ...


@dataclass(frozen=True)
class ArchiveEntry:
    key: str
    size: int
    sha256: str


def load_entry(manifest_path: Path) -> ArchiveEntry:
    document = json.loads(manifest_path.read_text(encoding="utf-8"))
    item = document["objects"][0]
    return ArchiveEntry(item["key"], item["size"], item["sha256"])


def safe_filename(key: str) -> str:
    path = PurePosixPath(key)
    if path.is_absolute() or ".." in path.parts or not path.name:
        raise ValueError("unsafe archive key")
    return path.name


def fetch_verified(reader: ObjectReader, entry: ArchiveEntry, target: Path) -> Path:
    target.mkdir(parents=True, exist_ok=True)
    final_path = target / safe_filename(entry.key)
    temporary_path = final_path.with_suffix(final_path.suffix + ".partial")
    digest = hashlib.sha256()
    written = 0

    try:
        with reader.open(entry.key) as source, temporary_path.open("wb") as output:
            while chunk := source.read(1024 * 1024):
                output.write(chunk)
                digest.update(chunk)
                written += len(chunk)
            output.flush()
            os.fsync(output.fileno())

        if written != entry.size:
            raise ValueError("archive size does not match the manifest")
        if digest.hexdigest() != entry.sha256:
            raise ValueError("archive digest does not match the manifest")
        temporary_path.replace(final_path)
        return final_path
    except BaseException:
        temporary_path.unlink(missing_ok=True)
        raise
```

Do not expand this into a universal database restore routine. Each database and file format has its own coherence and import rules. The script's job is to make an incorrect or incomplete archive stop before that irreversible application-specific step.

Exercise it in two ways. In automated tests, return truncated bytes, changed bytes, and unsafe keys from a test reader. In a scheduled recovery drill, use a fresh destination and a recovery identity that is distinct from production, retrieve a retained artifact, run the actual importer, and query a known record or application invariant. Logs should identify the manifest and object key without exposing signed URLs or credentials. Metrics should separate retrieval, validation, and import failures. Short messages matter here.

## Cost is a recovery-month calculation, not a storage-rate comparison

Object storage bills are workload equations. Retained byte-months are only one term; request classes, retrieval, data transfer, copies, and the cadence of verification can change the result. The AWS S3 pricing page illustrates the point by listing storage, requests, retrievals, and data transfer as distinct dimensions. It is an example of pricing structure, not a claim that every service exposes the same categories or rates.

Model two ordinary months and one recovery month. The recovery case should include retrieval of the retained set, interrupted transfers that may require retries, and temporary capacity for extraction or import. For a self-managed design, count replacement media, an actually independent copy, monitoring, certificate and identity maintenance, upgrades, and the time required to perform them. I'm not sure a "cheapest" answer means anything until these inputs and the recovery objective are written down; spare hardware and existing operational coverage can change the result, but uncounted work is still work.

The catch is that reducing verification to save requests may make the invoice calmer while eliminating the only evidence that recovery still works. A better compromise is a small frequent verification sample plus periodic full restores, with an explicit recovery-month budget. Measure the ordinary path and the emergency path separately.

## Rejected deployment and the condition that changes the choice

Reject the first design that places the application and its only object archive inside one failure domain. It asks a beginner to diagnose storage while recovering from the event that removed the application, and no signed URL can repair that boundary after the fact.

Self-managed storage is not suitable as a shortcut for a team without independent hardware, monitoring, patching capacity, and a practiced recovery procedure. It remains a valid choice when custody requirements demand controlled infrastructure or when a team already runs separate nodes and can demonstrate media replacement, replication, identity isolation, and restore drills. Cloud object storage is a poor fit when required custody or network constraints make its recovery objective unattainable. Choose the design whose documented boundaries and recovery evidence match the application's risk, then keep the manifest and restore contract portable enough to reassess later.

## References

- https://aws.amazon.com/s3/pricing/
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
