# Centralized Docker Log Retention & Retrieval System

> A documentation-only case study of a centralized logging and log-retention workflow built around Docker, Grafana Alloy, Grafana Loki, and Azure Blob Storage.

---

## Disclaimer

> This repository contains sanitized documentation and conceptual examples derived from a production logging project. Production infrastructure details, credentials, internal endpoints, and sensitive configuration are intentionally excluded. All hostnames, storage account names, and service names shown here are placeholders.

---

## 1. Project Overview

Docker services generate logs continuously for as long as the container is running. When a container is restarted, recreated, or replaced — which happens routinely during deployments, scaling events, or crashes — logs that were only available on the container's local filesystem or through `docker logs` can become difficult or impossible to retrieve.

This creates a practical problem for troubleshooting: if an issue is reported after a container has already been recreated, the logs needed to investigate it may no longer be accessible through normal Docker tooling.

**Objective:** Build a centralized logging workflow where service logs are collected, labeled, indexed, retained, and retrieved using service identity and time-based queries — independent of the lifecycle of any individual container.

This repository documents that workflow: why it was needed, how it was designed, how it was tested, and what was learned while implementing it.

---

## 2. Architecture

```text
Docker Services
      |
      v
Grafana Alloy
      |
      v
Grafana Loki
      |
      +--------------------+
      |                    |
      v                    v
Short/Active Retention   Azure Blob Storage
                               |
                               v
                        Long-Term Storage
      |
      v
Service Logs Backend
      |
      v
Service Logs Frontend
      |
      v
Developer / Authorized User
```

**Component summary:**

- **Docker Services** — the applications and services producing logs.
- **Grafana Alloy** — collects logs from Docker and forwards them, attaching identifying metadata (labels) along the way.
- **Grafana Loki** — receives, indexes, and stores the labeled log streams, and exposes a query API.
- **Short/Active Retention** — the recent window of logs kept readily queryable inside Loki's own storage.
- **Azure Blob Storage** — used for longer-term retention of log data beyond the active retention window.
- **Service Logs Backend** — a custom application layer that queries Loki on behalf of users.
- **Service Logs Frontend** — a UI that lets developers search, filter, and download logs by service and time range.
- **Developer / Authorized User** — the consumer of the system, retrieving historical logs without needing direct access to Loki or the underlying containers.

A more detailed, diagram-based explanation is available in [`docs/architecture.md`](docs/architecture.md).

---

## 3. Key Features

- Centralized Docker log collection across services
- Service-based log identification (via labels/metadata)
- Timestamp-based retrieval and filtering
- Historical log access after container restart or recreation
- Configurable retention behavior
- Long-term object storage for logs beyond the active retention window
- Developer-friendly log retrieval through a dedicated UI
- Validation of log continuity across container restarts/recreation
- Health and readiness checks for the logging pipeline itself

---

## 4. Technology Stack

| Component      | Technology          | Purpose                        |
| --------------- | -------------------- | -------------------------------- |
| Containers      | Docker                | Application workloads            |
| Collector       | Grafana Alloy         | Log collection and forwarding    |
| Log backend     | Grafana Loki 3.7.3    | Centralized log storage/query    |
| Object storage  | Azure Blob Storage    | Long-term log storage            |
| Frontend        | Service Logs UI       | Log discovery/retrieval          |
| Backend         | Service Logs API      | Query/download layer             |
| Reverse proxy   | Nginx                 | Routing/exposure of the UI/API   |

---

## 5. Log Flow

```text
Container
   ↓
Docker log files
   ↓
Grafana Alloy
   ↓
Labels / metadata
   ↓
Loki
   ↓
Retention / Object Storage
   ↓
Query
   ↓
Service Logs UI
```

Service labels and timestamps are what make historical retrieval possible. Without a consistent `service_name` label (and similar metadata such as container name, host, and time), there is no reliable way to ask "show me what this service logged last Tuesday" once the original container is gone. Labels turn a stream of raw text into something that can be filtered and queried; timestamps turn it into something that can be scoped to a specific incident window.

A deeper explanation of this flow, including how metadata is attached, is in [`docs/log-flow.md`](docs/log-flow.md).

---

## 6. Example Query

All examples below use placeholders and are **not** production endpoints.

Query logs for a given service:

```bash
curl -G -s "http://<LOKI_HOST>:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={service_name="<SERVICE_NAME>"}' \
  --data-urlencode 'limit=100'
```

Discover available service labels (useful for confirming what a Loki instance currently knows about):

```bash
curl -s "http://<LOKI_HOST>:3100/loki/api/v1/label/service_name/values"
```

> These are sanitized examples for illustration only. No real service names, hostnames, or endpoints from any production environment are included.

---

## 7. My Role

> Worked on the implementation, configuration, testing, troubleshooting, and documentation of a centralized Docker log retention workflow using Grafana Alloy and Grafana Loki, with object storage considered/used for longer-term retention.

This work was carried out as part of a DevOps Trainee role, with a focus on hands-on implementation and validation rather than end-to-end architectural ownership.

---

## 8. Documentation Index

| Document | Description |
| --- | --- |
| [`docs/architecture.md`](docs/architecture.md) | Detailed component breakdown and architecture diagram |
| [`docs/implementation.md`](docs/implementation.md) | Implementation phases from log collection to object storage |
| [`docs/log-flow.md`](docs/log-flow.md) | How a log line travels from container to UI |
| [`docs/retention.md`](docs/retention.md) | Retention concepts, short vs. long-term storage |
| [`docs/testing-validation.md`](docs/testing-validation.md) | Testing methodology used to validate the system |
| [`docs/troubleshooting.md`](docs/troubleshooting.md) | Real issues encountered and how they were diagnosed |
| [`docs/security.md`](docs/security.md) | Security practices and a pre-publish checklist |
| [`docs/production-considerations.md`](docs/production-considerations.md) | What's implemented/tested vs. future improvements |
| [`diagrams/architecture.md`](diagrams/architecture.md) | Standalone Mermaid architecture diagram |
| [`screenshots/README.md`](screenshots/README.md) | Guidance on what screenshots to include and how to sanitize them |

---

## 9. Scope of This Repository

This is a **documentation and portfolio/case-study repository**. It does not contain:

- Production Docker Compose files
- Production Grafana Alloy or Loki configuration
- Azure Storage account details, keys, or SAS tokens
- Real hostnames, IP addresses, or internal domains
- Real log content or customer data

Where configuration or commands are shown, they are sanitized examples intended to illustrate concepts, not to be run against real infrastructure as-is.
