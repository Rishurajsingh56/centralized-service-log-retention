# Troubleshooting

This document collects realistic troubleshooting scenarios encountered while implementing and testing the logging pipeline, along with how they were diagnosed.

---

## Docker Container Reported as Unhealthy

When a container shows as `unhealthy`, don't assume the application itself is broken — the healthcheck configuration is just as likely to be the problem.

Steps to investigate:

1. Check the Docker health status:

   ```bash
   docker inspect <CONTAINER> --format='{{json .State.Health}}'
   ```

2. Review the healthcheck logs included in that output — they show the actual command that was run and its result.
3. Manually verify the endpoint the healthcheck is supposed to be checking, from inside or alongside the container.
4. Confirm that the command the healthcheck relies on (e.g., a shell, `curl`, `wget`) actually exists inside that specific image.

## Loki Readiness vs. Docker Healthcheck Disagreement

```bash
curl -i http://<LOKI_HOST>:3100/ready
```

It's possible for this to return `200 OK` (Loki is, in fact, ready) while Docker simultaneously reports the container as unhealthy. This doesn't mean Loki is broken — it usually means the Docker healthcheck itself is misconfigured (see the next section).

## Healthcheck Binary Mismatch

**This was one of the more important lessons from this project.**

Minimal/slim container images often don't include a full shell or common utilities. If a healthcheck is defined assuming the presence of:

- `/bin/sh`
- `wget`
- `curl`

...but the image doesn't actually contain one of those, the healthcheck itself will fail to execute — regardless of whether the application is working correctly. Docker will then report the container as unhealthy, even though the application is serving requests normally.

**Takeaway:** healthchecks must use commands that are actually available inside the specific image being used, not commands assumed to be present based on general familiarity with Linux containers.

## No Logs Appearing

If expected logs aren't showing up in Loki, check each stage of the pipeline in order:

1. **Container is generating logs** — confirm with `docker logs <CONTAINER>` that the application is actually producing output.
2. **Alloy is running** — confirm the Alloy process/container itself is up.
3. **Alloy can access the log source** — confirm Alloy has the access it needs to read Docker's logs (permissions, socket access, etc., depending on collection method).
4. **Loki is reachable** — confirm Alloy can reach Loki's push endpoint over the network.
5. **Labels are correct** — confirm the expected `service_name`/`container_name` labels are actually being attached (see the label-listing check in [`docs/testing-validation.md`](testing-validation.md)).
6. **Retention/storage is functioning** — confirm logs aren't being dropped or misrouted due to a retention or storage configuration issue.

Working through these in order usually narrows the problem down quickly, since each stage depends on the one before it.

## Query Exceeds Retention/Query Limits

Loki deployments are often configured with limits on how far back a query can look, or how large a query result can be. If a query for older logs comes back empty or errors out, it's worth checking whether the requested time range falls outside the configured retention or query-limit window before assuming the data is missing entirely.

No specific limit values are documented here, since these are environment-specific and not something that should be assumed without checking the actual configuration in place.

---

## Practical Troubleshooting Lesson

The clearest lesson from this project can be summarized simply:

> A container can be fully operational — serving requests successfully — while Docker still reports it as unhealthy, because the healthcheck itself is incorrectly configured rather than the application being broken.

Example of the disconnect:

```text
Application:
HTTP /ready → 200 OK

Docker healthcheck:
Command unavailable → unhealthy
```

**The practical takeaway:** health status should always be investigated from both sides:

1. **Application readiness** — is the application itself actually responding correctly?
2. **Healthcheck execution** — is the healthcheck command itself able to run successfully inside that container's image?

Treating an "unhealthy" status as automatically meaning "the application is broken" can send troubleshooting in the wrong direction. This is presented here as a technical learning from the project, not as a failure of the system.
