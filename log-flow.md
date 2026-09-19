# Log Flow

This document walks through the journey of a single log line, from the moment it is written by a container to the moment a developer views it in the Service Logs UI.

---

## The Flow

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

## Step by Step

### 1. Container writes a log line

An application inside a Docker container writes a line to stdout or stderr, as it normally would.

### 2. Docker captures it

Docker's logging driver captures that output and makes it available through Docker's own log storage mechanism (e.g., `docker logs`, or the underlying log files depending on the driver in use).

### 3. Grafana Alloy collects it

Alloy, running alongside the Docker services, picks up the new log line. At this point the log line is still just raw text tied to a specific container.

### 4. Labels and metadata are attached

Alloy attaches metadata to the log line before forwarding it, most importantly:

- `service_name`
- `container_name`
- `host`
- `timestamp`

This step is what transforms a raw log line into something that can later be filtered and queried independently of which specific container instance produced it.

### 5. Loki receives and indexes it

Loki stores the log line and indexes it by its labels. The log content itself is stored efficiently alongside the index, but querying is driven primarily by labels plus time range.

### 6. Retention/storage tiering applies

Depending on age, the log either remains in Loki's active storage or — once retention rules apply — becomes eligible to be moved to Azure Blob Storage for longer-term retention.

### 7. A query is made

When a developer needs to investigate something, the Service Logs Backend issues a query to Loki (e.g., `{service_name="<SERVICE_NAME>"}` filtered to a time range), and, where relevant, checks the long-term store for older data.

### 8. Results are displayed

The Service Logs Frontend renders the returned log lines, letting the user search within them, adjust the date range, and download the results if needed.

---

## Why Labels and Timestamps Matter

Two pieces of metadata make this whole flow useful:

- **Service labels** (`service_name`, `container_name`) let logs be found by *what produced them*, independent of container restarts or recreation. Without this, a log line is just anonymous text with no reliable way to associate it back to a service once the original container is gone.
- **Timestamps** let logs be scoped to *when something happened*. Most real investigations start with a time window ("this started failing around 2pm") rather than a full-text search, so timestamp-based filtering is often the first and most effective filter applied.

Together, these two pieces of metadata are what allow the system to answer the core question it was built for: *"What did this service log during this time window, even if the container that produced it no longer exists?"*
