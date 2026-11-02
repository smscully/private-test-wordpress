# ADR 0005: Loki and Grafana Alloy for logs

- **Status:** Accepted
- **Date:** 2026-11-01
- **Applies to:** both sites

## Context
The design covered metrics (Prometheus), alerts (Alertmanager to Slack), network flows (Hubble), security events (Falco) and optional traces (Tempo), but had **no central log storage**. Logs could only be read live with `kubectl logs` or `stern`; once a pod restarted, was evicted, or a staging run was deleted, its logs were lost. That makes debugging and incident review much harder.

Constraints: box 1 of the home lab has 8 GB of RAM and already uses about 6–7 GB; both sites should work the same way; the home site runs outside AWS; cost must stay near zero.

## Decision
- **Grafana Loki** in **single-binary mode** stores logs, with its data in an **S3 bucket** per site and a small local cache. Retention: **7 days** at home, **14 days** for russetleaf.com (bucket in the always-on edge stack, so demo logs outlive the demo).
- **Grafana Alloy** runs on every node, collects container logs, applies a **small fixed label set** (`site`, `env`, `cluster`, `namespace`, `app`, `container`), drops noise and masks obvious secrets.
- **Talos system logs** are sent by Talos to Alloy, so node-level logs are searchable too.
- **Loki's ruler** sends log-based alerts to the existing Alertmanager → Slack routing.
- The **EKS control plane audit logs** stay in CloudWatch Logs.

## Alternatives considered
- **VictoriaLogs:** lighter on memory than Loki and simple to run; a reasonable fit for 8 GB. Not chosen because Loki is more widely used, integrates natively with Grafana (correlation with metrics and traces), and is the more recognisable skill.
- **OpenSearch / ELK (Elasticsearch, Logstash, Kibana):** full-text search over everything, but needs several GB of memory; far too heavy for this hardware.
- **CloudWatch Logs only:** fine on AWS, but doesn't cover the home site, and costs per GB ingested.
- **Promtail** as the collector: deprecated in favour of Alloy.
- **No central logs (status quo):** rejected; losing logs on every restart is not production-grade.

## Consequences
- Logs survive restarts and deletions, and are searchable alongside metrics and traces in Grafana.
- New alert type: log-based alerts with runbooks.
- Roughly **0.3–0.7 GB** more memory in use on box 1; retention, label count and dropped noise must be watched.
- Alloy needs a narrow Kyverno `PolicyException` (host paths for log files, a host port for Talos logs).
- Logs may hold personal data (IP addresses, occasionally emails): short retention, masking and restricted Grafana access are required.

## When to revisit
- Box 1's memory becomes a blocker: consider VictoriaLogs, or shorter retention.
- Log volume grows enough that Loki's single-binary mode struggles: move to its scalable deployment mode (on AWS) or reduce what's collected.
- A requirement for long-term log retention (e.g. an audit): add S3 lifecycle transitions to cheaper storage classes.

## Open items
- Home plan step 4.6: record actual memory use and daily log volume after a week.
- AWS plan step 2.4: confirm the 14-day retention is enough for reviewing demos.
