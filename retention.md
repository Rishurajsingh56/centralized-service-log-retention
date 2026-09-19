# Retention

This document explains the retention approach used in the system: why it's needed, how short-term and long-term storage differ, and what factors affect how much storage retention actually requires.

---

## Why Retention Is Needed

Without a retention strategy, a centralized logging system has two failure modes:

1. **Retain everything forever** — storage grows without bound, and query performance in the "hot" storage tier degrades as the index grows.
2. **Retain too little** — logs needed for an investigation may already be gone by the time someone asks for them.

A deliberate retention policy balances these two concerns: keep enough recent data readily queryable, and move older data somewhere cheaper without deleting it outright (where that's warranted).

---

## Short-Term vs. Long-Term Storage

- **Short-term / active retention** — logs kept directly in Loki's own storage. This is optimized for speed: recent logs are the ones most frequently queried, typically during active troubleshooting.
- **Long-term retention** — older logs moved to Azure Blob Storage. Queried less often, but retained for cases where a historical investigation is needed (e.g., "what did this service do a few weeks ago").

## Retention Duration — 15 vs. 30 Days (Concept)

Retention duration is a trade-off, not a fixed rule. As an illustrative example:

- A **15-day** active retention window keeps recent, high-value logs fast to query, with a smaller storage footprint.
- A **30-day** window gives more breathing room for delayed investigations, at the cost of roughly double the active storage (all else being equal).

The right duration depends on how often logs older than the active window are actually needed, and how expensive active storage is compared to long-term object storage.

---

## Why Log Volume Is the Main Cost Driver

Retention duration alone doesn't determine cost — log **volume** does. A service that logs verbosely will consume far more storage over the same retention period than a quiet service. This means:

- Noisy or overly verbose logging directly increases storage cost.
- Retention policy decisions should be made alongside a review of what's actually being logged (and at what log level).

## Why Object Storage Is Useful for Historical Logs

Azure Blob Storage is well suited for the long-term tier because:

- It's significantly cheaper per GB than keeping data in Loki's active storage indefinitely.
- Historical logs are accessed far less frequently, so the trade-off of slightly slower retrieval is acceptable.
- It decouples "how long can we afford to keep logs" from "how fast does routine troubleshooting need to be."

---

## Estimating Storage Requirements

A simple starting formula for estimating storage needs:

```text
Required Storage ≈ Average Log Volume per Day × Retention Days
```

This is a **starting point**, not a precise sizing formula. Actual consumption is affected by:

- **Compression** — Loki compresses log chunks, which can significantly reduce the effective storage footprint versus raw log volume.
- **Indexing overhead** — the index itself consumes additional space beyond the raw log content.
- **Replication** — if the storage backend replicates data for durability, effective usage is higher than the raw retained volume.
- **Query/storage architecture** — how chunks are structured and stored can affect both cost and query performance in ways a simple volume × days formula doesn't capture.

Any actual production log volume figures are intentionally excluded from this repository.

---

## Validating Retention Policies

Retention policies shouldn't be assumed to work correctly just because they're configured — they should be validated. At minimum:

1. Confirm that logs older than the configured active retention window are no longer served from active storage (or are correctly redirected to long-term storage, depending on design).
2. Confirm that logs within the retention window remain fully queryable, including after container restarts/recreation.
3. Periodically check actual storage growth against the estimate to catch unexpectedly high log volume early.

See [`docs/testing-validation.md`](testing-validation.md) for the testing methodology used to validate parts of this system.
