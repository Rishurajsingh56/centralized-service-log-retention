# Production Considerations

This document distinguishes between what was actually implemented and tested as part of this project, and what remains a future improvement. Nothing here is claimed as implemented unless there is evidence for it from the implementation and testing work described elsewhere in this repository.

---

## Implemented / Tested

- **Docker log collection via Grafana Alloy** — validated that Alloy discovers and forwards logs from running Docker services.
- **Centralized storage and query via Grafana Loki 3.7.3** — validated that logs are indexed and queryable by service label and time range.
- **Service-based labeling** — validated that `service_name`/`container_name` labels are correctly attached and queryable.
- **Log persistence across container restart/recreation** — explicitly tested (see [`docs/testing-validation.md`](testing-validation.md)) to confirm logs remain queryable after the originating container is gone.
- **Retrieval through a dedicated UI/backend** — validated that developers can search and download logs without direct Loki access.
- **Basic health/readiness checks** — validated Loki's `/ready` endpoint and Docker container health status as part of routine testing.
- **Long-term retention via Azure Blob Storage** — used as the long-term tier for logs beyond the active retention window, as described conceptually in [`docs/implementation.md`](implementation.md) and [`docs/retention.md`](retention.md).

## Future Improvement

The following areas are identified as valuable next steps, but were **not** fully implemented or validated as part of this project:

- **Automated monitoring and alerting** on the logging pipeline itself (e.g., alerting if Alloy stops forwarding logs, or if Loki becomes unreachable).
- **Formal backup/recovery procedures** for the logging pipeline's own state and configuration.
- **Access control refinement** — moving beyond basic restriction toward more granular, role-based access to the Service Logs UI.
- **Query performance tuning** at larger log volumes than were tested here.
- **Cost tracking/reporting** tied specifically to log storage growth over time.
- **Formal disaster-recovery testing** for the object storage tier.
- **Automated retention policy enforcement validation** (currently validated manually, as described in [`docs/retention.md`](retention.md)).

---

## Other Considerations Discussed (Not Necessarily Implemented)

These are topics worth being aware of for anyone extending this type of system, without a claim that they were specifically addressed here:

- **Disk usage** on hosts running Docker services and the logging stack, independent of Loki's own storage.
- **Log volume growth** over time as services are added or as logging verbosity changes.
- **Network connectivity** between Alloy, Loki, and the object storage backend, and what happens to log delivery if connectivity is temporarily lost.
- **Loki availability** — what a single point of failure in the logging pipeline would mean for observability during an incident.
- **Alloy reliability** — behavior under load or during restarts of the collector itself.

---

## Summary

| Area | Status |
| --- | --- |
| Log collection (Alloy) | Implemented / Tested |
| Centralized storage/query (Loki) | Implemented / Tested |
| Service-based labeling | Implemented / Tested |
| Restart/recreation log persistence | Implemented / Tested |
| UI-based retrieval | Implemented / Tested |
| Health/readiness checks | Implemented / Tested |
| Long-term storage (Azure Blob Storage) | Implemented / Tested |
| Monitoring & alerting on the pipeline | Future Improvement |
| Backup/recovery procedures | Future Improvement |
| Fine-grained access control | Future Improvement |
| Large-scale query performance tuning | Future Improvement |
| Cost tracking | Future Improvement |
