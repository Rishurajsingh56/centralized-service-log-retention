# Implementation

This document describes the implementation approach in phases, from log collection through to long-term object storage. Each phase is described conceptually — no production configuration values, real labels, or credentials are included.

---

## Phase 1 — Log Collection

Grafana Alloy was configured to discover Docker containers and collect their log output. Conceptually, this involves:

- Pointing Alloy at the Docker log source (e.g., the Docker socket or log files, depending on configuration approach).
- Letting Alloy discover running containers so new services are picked up automatically without manual per-container setup.
- Forwarding collected log lines toward Loki's push endpoint.

**Example (sanitized) of the kind of endpoint Alloy forwards to:**

```text
http://<LOKI_HOST>:3100/loki/api/v1/push
```

No real Alloy configuration file, discovery rules, or internal paths are included here, since these can reveal details about the production environment.

---

## Phase 2 — Metadata (Labels)

For logs to be useful once centralized, they need to carry identifying metadata. The labels attached during collection typically include:

- `service_name` — which application/service produced the log
- `container_name` — the specific container instance
- `host` — which host/node the container was running on
- `timestamp` — when the log line was generated

These labels are what later allow a query like "show me logs for `<SERVICE_NAME>` between two points in time" to work, even after the original container no longer exists.

**Important consideration:** Labels should be kept low-cardinality. Attaching something like a unique request ID or a rapidly changing value as a Loki label (rather than as log content) can significantly degrade Loki's index performance. This is a deliberate design choice, not an oversight.

Actual production label configuration is not included here, since label naming conventions can indirectly reveal internal service naming.

---

## Phase 3 — Grafana Loki

Loki (version 3.7.3) acts as the centralized log storage and query engine. Its responsibilities in this system:

- Accept pushed log streams from Alloy.
- Index logs by label (not full text), enabling efficient filtering.
- Store log content associated with those labels.
- Expose a query API (`/loki/api/v1/query_range`, label endpoints, etc.) for retrieval.
- Enforce configured retention and query-range limits.

Loki was treated as the "source of truth" for centralized logs within the active retention window, with the Service Logs Backend acting as the primary consumer of its query API.

---

## Phase 4 — Retention

Retention was handled conceptually as a two-tier model:

1. **Active retention** — recent logs kept directly in Loki's own storage for fast, routine querying.
2. **Long-term retention** — older logs moved to Azure Blob Storage, accessible when needed but not part of the default hot-path query.

See [`docs/retention.md`](retention.md) for a more detailed discussion of retention duration, storage growth, and validation.

---

## Phase 5 — Object Storage (Azure Blob Storage)

Azure Blob Storage was used as the long-term storage layer. Conceptually:

- Logs beyond the active retention window are persisted in Blob Storage rather than being discarded.
- This keeps Loki's active storage smaller and faster, while still preserving historical log data for less frequent, longer-range investigations.
- Access to this storage layer is restricted — see [`docs/security.md`](security.md).

No actual Azure Storage account names, container names, access keys, or SAS tokens are included in this repository. Any storage account or container names shown elsewhere in this documentation are placeholders such as `<AZURE_STORAGE_ACCOUNT>` and `<STORAGE_CONTAINER>`.

---

## Summary of Phases

| Phase | Focus | Status |
| --- | --- | --- |
| 1 | Log collection via Alloy | Implemented / Tested |
| 2 | Metadata/label attachment | Implemented / Tested |
| 3 | Centralized storage/query via Loki | Implemented / Tested |
| 4 | Retention (active tier) | Implemented / Tested |
| 5 | Long-term storage via Azure Blob Storage | Implemented / Tested (conceptually described here) |

For what remains a future improvement rather than something already validated, see [`docs/production-considerations.md`](production-considerations.md).
