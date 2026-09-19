# Centralized Docker Log Retention & Retrieval System

> A production-tested DevOps case study for centralized Docker service log collection, retention, search, and retrieval using **Grafana Alloy, Grafana Loki, Docker, and a custom Service Logs UI/API**.

<p align="center">

**Docker** • **Grafana Alloy** • **Grafana Loki** • **Nginx** • **Backend API** • **Frontend UI** • **Azure**

</p>

---

## 📌 About This Project

This project addresses a practical problem encountered while working with containerized services:

> **What happens to application logs when a Docker service/container is restarted, recreated, replaced, or removed?**

With normal container-level logging, logs can become difficult to retrieve after the original container lifecycle ends. This creates a problem when developers or operations teams need to investigate an incident that occurred earlier.

The solution explored and implemented a **centralized service log retention and retrieval workflow** where logs are collected independently of an individual container's lifecycle and can be retrieved using:

* Server
* Service
* Date/time range
* Service labels and metadata

The project was **implemented and tested in a real server environment** as part of hands-on DevOps work.

The public GitHub repository intentionally contains **sanitized documentation, architecture diagrams, conceptual examples, and selected screenshots instead of production source code/configuration**, because the original implementation belongs to a production environment.

---

## 🎯 Problem Statement

Docker services continuously generate logs while containers are running.

A typical workflow may look like:

```text
Application
     ↓
Docker Container
     ↓
docker logs
```

The problem appears when the container is:

```text
Restarted
   ↓
Recreated
   ↓
Replaced
   ↓
Removed
```

At that point, depending on the logging setup, previously available logs may no longer be conveniently accessible through the original container.

For troubleshooting, this creates a practical question:

> **How can developers retrieve historical service logs even after the original container no longer exists?**

---

## 💡 Solution

The project introduces a centralized logging workflow:

```text
┌─────────────────────┐
│   Docker Services   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│    Grafana Alloy    │
│  Log Collection     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│    Grafana Loki     │
│ Storage + Query API │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Service Logs API  │
│ Backend / Query     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Service Logs UI   │
│ Search / Filter /   │
│ View / Download     │
└──────────┬──────────┘
           │
           ▼
     Authorized User
```

This separates log availability from the lifecycle of individual Docker containers.

---

# 🏗️ Architecture

```text
                         ┌──────────────────────┐
                         │    Docker Services   │
                         │   Multiple Services  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    Grafana Alloy     │
                         │ Log Collection       │
                         │ Labels / Metadata    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     Grafana Loki     │
                         │ Log Storage & Query  │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                │
                    ▼                                ▼
          ┌──────────────────┐             ┌──────────────────┐
          │ Active Retention │             │ Long-Term Tier   │
          │ / Queryable Logs │             │ Object Storage   │
          └──────────────────┘             └──────────────────┘

                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  Service Logs API    │
                         │ Backend              │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  Service Logs UI     │
                         │ Frontend             │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Developer /          │
                         │ Authorized User      │
                         └──────────────────────┘
```

### Core Components

| Component                 | Role                                                                        |
| ------------------------- | --------------------------------------------------------------------------- |
| **Docker**                | Runs application and service workloads                                      |
| **Grafana Alloy**         | Collects Docker logs and attaches useful metadata                           |
| **Grafana Loki**          | Centralized log storage and query layer                                     |
| **Service Logs Backend**  | Application API responsible for log retrieval and filtering                 |
| **Service Logs Frontend** | User-facing interface for discovering, viewing and downloading logs         |
| **Nginx**                 | Routing/exposure layer for the application                                  |
| **Object Storage**        | Considered as the long-term retention layer                                 |
| **Authorized User**       | Retrieves historical service logs without requiring direct container access |

---

# 🔄 End-to-End Log Flow

```text
Docker Container
       │
       ▼
Docker Log Files
       │
       ▼
Grafana Alloy
       │
       ├── service_name
       ├── container metadata
       ├── host metadata
       └── timestamps
       │
       ▼
Grafana Loki
       │
       ▼
Retention / Historical Storage
       │
       ▼
Backend API
       │
       ▼
Frontend UI
       │
       ▼
Search / Filter / View / Download
```

The important part of the design is the use of **consistent service identity and timestamps**.

Instead of depending on a specific container ID, logs can be associated with the service that produced them.

This makes it possible to investigate historical activity even when the original container has been recreated.

---

# 🖥️ Service Logs UI

A dedicated frontend was developed for the log retrieval workflow.

The UI is designed around the actual operational requirement rather than exposing the underlying logging infrastructure directly to developers.

### Main workflow

```text
Select Server
     ↓
Select Service
     ↓
Select Date / Time Range
     ↓
Retrieve Logs
     ↓
View Historical Logs
     ↓
Download Logs
```

### UI capabilities

* Server-based log discovery
* Service-based filtering
* Date/time filtering
* Historical log retrieval
* Log viewing
* Log download workflow
* Backend-driven data retrieval

### 📸 UI Preview

> Screenshots can be added here after sanitizing server names, domains, IP addresses and other internal information.

```text
screenshots/
├── service-logs-dashboard.png
├── service-selection.png
├── historical-logs.png
└── log-download.png
```

---

# ⚙️ Backend / API

A custom backend/API layer was developed to sit between the frontend and centralized logging system.

Instead of giving developers direct access to Loki or Docker infrastructure:

```text
Developer
    ↓
Frontend
    ↓
Backend API
    ↓
Loki
```

The backend handles the application-side workflow for:

* Server discovery
* Service discovery
* Log retrieval
* Date/time filtering
* Query handling
* Log response processing
* Download workflow
* Communication with the frontend

This creates a cleaner separation between the **user-facing application** and the underlying logging infrastructure.

---

# 🔎 Log Retrieval

The system uses Loki labels and timestamps to identify and retrieve service logs.

Example sanitized query:

```bash
curl -G -s "http://<LOKI_HOST>:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={service_name="<SERVICE_NAME>"}' \
  --data-urlencode 'limit=100'
```

Service label discovery:

```bash
curl -s "http://<LOKI_HOST>:3100/loki/api/v1/label/service_name/values"
```

All endpoints, hostnames and service names shown in this repository are placeholders.

---

# 🧪 Testing & Validation

The workflow was tested against the actual logging requirements rather than only being documented conceptually.

### Validation included

```text
Service generates logs
        ↓
Alloy collects logs
        ↓
Loki receives logs
        ↓
Service labels verified
        ↓
Logs queried
        ↓
Container/service lifecycle changed
        ↓
Historical logs checked
        ↓
Logs retrieved through application workflow
```

The testing focused on validating that logs remain retrievable independently of the lifecycle of the original container.

Additional troubleshooting and validation details are documented in:

* [`docs/testing-validation.md`](docs/testing-validation.md)
* [`docs/troubleshooting.md`](docs/troubleshooting.md)

---

# 🧑‍💻 My Role

This project involved hands-on work across the logging infrastructure and the application layer.

### Infrastructure / DevOps

* Studied the existing logging requirement and operational problem.
* Worked on centralized Docker log collection.
* Configured and worked with Grafana Alloy.
* Worked with Grafana Loki for centralized log storage and querying.
* Worked with service labels and metadata for reliable log identification.
* Tested historical log retrieval.
* Tested behavior across container/service restart and recreation scenarios.
* Troubleshot logging and retrieval issues.
* Worked on retention-related requirements.
* Documented the implementation, testing process and troubleshooting findings.

### Backend

* Contributed to the implementation of the Service Logs backend/API.
* Implemented the application workflow for retrieving logs from the centralized logging layer.
* Worked with server, service and date/time based filtering.
* Integrated backend functionality with the frontend.
* Tested API responses and log retrieval behavior.

### Frontend

* Contributed to the implementation of the Service Logs frontend.
* Built the user workflow for:

  * Server selection
  * Service selection
  * Date/time filtering
  * Log viewing
  * Log retrieval
  * Download workflow
* Integrated the UI with the backend API.
* Tested the complete frontend-to-backend-to-Loki workflow.

### AI-Assisted Development

**Claude was used as an AI-assisted development tool** during frontend and backend development.

It was used to help with:

* UI implementation
* Backend development assistance
* Code structure and implementation ideas
* Debugging assistance
* Refinement of the application workflow

The resulting implementation was **integrated, tested, debugged and validated in the target environment** rather than being presented as AI-generated code alone.

### Documentation

I also documented:

* Architecture
* Log flow
* Implementation approach
* Retention behavior
* Testing methodology
* Troubleshooting
* Security considerations
* Production considerations

---

# 🛡️ Production Safety & Public Repository Scope

This repository is intentionally a **documentation and portfolio/case-study repository**.

The actual production environment contains configuration and infrastructure information that should not be publicly exposed.

Therefore, the public repository intentionally does **not** contain:

* Production Docker Compose files
* Production Grafana Alloy configuration
* Production Loki configuration
* Azure Storage account credentials
* SAS tokens
* Access tokens
* Passwords
* Private keys
* Real IP addresses
* Internal domains
* Production hostnames
* Customer information
* Real production log content
* Sensitive environment variables

Instead, the repository contains:

```text
Architecture
    +
Documentation
    +
Sanitized examples
    +
Conceptual configurations
    +
Testing methodology
    +
Troubleshooting notes
    +
Selected sanitized screenshots
```

This allows the project to demonstrate the **technical approach and engineering work without exposing production infrastructure or credentials**.

---

# ☁️ Long-Term Retention

The architecture considers object storage as the longer-term retention tier.

Conceptually:

```text
                 Recent Logs
                     │
                     ▼
               Grafana Loki
                     │
                     ▼
             Active Retention
                     │
                     ▼
             Object Storage
                     │
                     ▼
            Long-Term Retention
```

Where Azure Blob Storage is referenced in this repository, it should be understood as the **long-term storage architecture/retention layer** and not as a disclosure of any production storage account or credentials.

---

# 🧰 Technology Stack

| Area                   | Technology                          |
| ---------------------- | ----------------------------------- |
| Container Platform     | Docker                              |
| Log Collector          | Grafana Alloy                       |
| Log Storage / Query    | Grafana Loki 3.7.3                  |
| Backend                | Service Logs API                    |
| Frontend               | Service Logs UI                     |
| Reverse Proxy          | Nginx                               |
| Long-Term Storage      | Azure Blob Storage / Object Storage |
| Development Assistance | Claude AI-assisted development      |

---

# 📂 Repository Structure

```text
centralized-service-log-retention/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── implementation.md
│   ├── log-flow.md
│   ├── retention.md
│   ├── testing-validation.md
│   ├── troubleshooting.md
│   ├── security.md
│   └── production-considerations.md
│
├── diagrams/
│   └── architecture.md
│
└── screenshots/
    ├── README.md
    └── ...
```

The repository is structured to keep the public case study understandable without exposing production implementation details.

---

# 📚 Documentation

| Document                                                                 | Purpose                                           |
| ------------------------------------------------------------------------ | ------------------------------------------------- |
| [`docs/architecture.md`](docs/architecture.md)                           | System architecture and components                |
| [`docs/implementation.md`](docs/implementation.md)                       | Implementation approach                           |
| [`docs/log-flow.md`](docs/log-flow.md)                                   | End-to-end log movement                           |
| [`docs/retention.md`](docs/retention.md)                                 | Retention and storage concepts                    |
| [`docs/testing-validation.md`](docs/testing-validation.md)               | Testing and validation methodology                |
| [`docs/troubleshooting.md`](docs/troubleshooting.md)                     | Issues encountered and troubleshooting            |
| [`docs/security.md`](docs/security.md)                                   | Security and public repository checklist          |
| [`docs/production-considerations.md`](docs/production-considerations.md) | Production considerations and future improvements |
| [`diagrams/architecture.md`](diagrams/architecture.md)                   | Standalone architecture diagram                   |

---

# 🚀 Key Takeaways

This project provided hands-on experience with a complete logging workflow:

```text
Problem Identification
        ↓
Architecture Study
        ↓
Docker Log Collection
        ↓
Grafana Alloy
        ↓
Grafana Loki
        ↓
Retention & Historical Retrieval
        ↓
Backend API
        ↓
Frontend UI
        ↓
Testing & Troubleshooting
        ↓
Documentation & Security Sanitization
```

The main objective was not simply to collect logs, but to make **historical service logs accessible and useful for troubleshooting even when individual containers change over time**.

---

## 🔐 Final Note

> This repository is intentionally sanitized for public sharing. It demonstrates the architecture, implementation approach, engineering workflow, testing methodology and lessons learned from the project while keeping production infrastructure, credentials, internal configuration and sensitive data private.

**Built, implemented, tested and documented as a hands-on DevOps project.**
