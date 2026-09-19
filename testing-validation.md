# Testing & Validation

This document describes the testing methodology used to validate the logging and retention pipeline. All examples use non-production/test containers and sanitized hostnames.

---

## 1. Service Health

Start by confirming the state of the relevant Docker services:

```bash
docker ps
```

Check the reported status for each service:

- **running** — container is up and operating normally.
- **healthy** — container is up and its configured healthcheck is passing.
- **unhealthy** — container is up but its healthcheck is failing (see [`docs/troubleshooting.md`](troubleshooting.md) for why this doesn't always mean the application itself is broken).
- **restarting** — container is crash-looping or being restarted, worth investigating before testing further.

## 2. Loki Readiness

Confirm Loki itself is ready to accept writes and serve queries:

```bash
curl -i http://<LOKI_HOST>:3100/ready
```

Expected response:

```text
HTTP/1.1 200 OK
ready
```

## 3. Verify Available Service Labels

Confirm that Loki is actually receiving labeled data from Alloy by checking which service labels currently exist:

```bash
curl -s "http://<LOKI_HOST>:3100/loki/api/v1/label/service_name/values"
```

If the expected service isn't present here, that points to a collection or forwarding issue rather than a query issue — see the troubleshooting guide.

## 4. Generate Test Logs

Testing should use a dedicated, non-production/local test container rather than a production service:

```bash
docker run --name log-test alpine sh -c \
  'for i in $(seq 1 20); do echo "test log $i"; sleep 1; done'
```

This produces a small, predictable set of log lines that can be searched for later to confirm the pipeline is working end to end.

> This is a local/test example only. It should not be run against, or substituted for, a production service.

## 5. Restart / Recreation Test

A core requirement of this system is that logs remain queryable even after the container that produced them is gone. To validate this:

1. Generate test logs as above.
2. Confirm the logs are queryable in Loki.
3. Stop and remove the test container (or recreate it).
4. Re-query Loki for the same `service_name`/`container_name` and time range.
5. Confirm the previously generated logs are still returned.

This test directly validates the core value proposition of the system: log availability independent of container lifecycle.

> Do not perform restart/recreation testing against production services. Use a dedicated test container as shown above.

## 6. Verify Through the UI

Finally, validate the same flow through the Service Logs Frontend rather than raw API calls:

1. Select the test service.
2. Choose a date range covering the test window.
3. Search for a known string from the generated test logs.
4. Confirm the expected log lines appear.
5. Confirm the download functionality returns the expected content.

---

## Summary

| Test | Validates |
| --- | --- |
| `docker ps` | Container health/status |
| `/ready` check | Loki availability |
| Label listing | Alloy → Loki collection is working |
| Test log generation | End-to-end pipeline functions |
| Restart/recreation test | Logs persist independent of container lifecycle |
| UI verification | Full user-facing flow works as expected |

This methodology was used to validate the pipeline in a safe, non-production-impacting way before relying on it for real troubleshooting scenarios.
