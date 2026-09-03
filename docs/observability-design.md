# ProjectBoard Observability Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Security | [Security Architecture](security-architecture.md) |
| Network | [Network Design](network-design.md) |
| Last updated | 2026-09-03 |
This document defines Version 1 metrics, technical logs, health, dashboards, alerting, and correlation.
Business Audit, messaging semantics, backup implementation, and firewall rules belong to their dedicated designs.

# 1. Principles
1. Observability is centralized where practical.
2. Monitoring failure shall not block business operations.
3. Business Audit and technical logs remain separate.
4. Telemetry identifies environment and component.
5. Relevant operations propagate `correlationId`.
6. High-cardinality values are not metric labels.
7. Secrets and authentication tokens are not logged.
8. Collection remains outside the synchronous business path.
9. Monitoring access does not imply administrative access.
10. Observability is introduced progressively as infrastructure is built.

# 2. Monitoring Platform
The dedicated `monitoring` VM hosts:
```text
Prometheus
Grafana
Loki
Grafana Alloy
Alertmanager
````

Exporters and source-side Alloy collectors may run on other hosts.
The Monitoring VM is an accepted single observability failure domain in Version 1; monitoring HA is not required.

# 3. Metrics

Prometheus primarily uses pull-based collection:

```text
Prometheus → scrape targets
```

Targets may include Backend, Gateway, PostgreSQL, RabbitMQ, Redis, MinIO, Linux hosts, Keycloak, Vault, and later Kubernetes.
Applications shall not synchronously push metrics to Prometheus during request processing.
Spring Boot uses Actuator/Micrometer-compatible instrumentation.
Useful application signals include:

* request rate, status, and latency;
* JVM/process health and database-pool state;
* Outbox pending count, oldest pending age, and publication failures;
* messaging consumer failures.

# 4. Technical Logs
Logs primarily follow:

```text
Source logs → source-side Alloy → Loki → Grafana
```

Source-side Alloy may collect stdout, journal, container, or host logs; exact placement remains open.
Logging shall not synchronously wait for Loki.
Useful structured fields include timestamp, level, environment, service, component, `correlationId`, request method/route, status, and relevant `eventId`/`eventType`.
High-cardinality values such as `correlationId`, `eventId`, user IDs, and object IDs remain searchable fields rather than primary Loki labels.

# 5. Correlation
Relevant flows should propagate:

```text
HTTP Request
→ Backend
→ Outbox
→ RabbitMQ
→ Consumer
→ Activity/Audit
```

`correlationId` identifies a related operation; `eventId` identifies one asynchronous event.
Caller-provided correlation values are untrusted and shall be validated or replaced at the trusted boundary.

# 6. Health Model
ProjectBoard distinguishes:

* **Liveness:** can the process continue running?
* **Readiness:** can the instance safely receive normal traffic?
* **Dependency health:** diagnostic status of external dependencies.

| Dependency       | Liveness                                  | Readiness / Effect                                 |
| ---------------- | ----------------------------------------- | -------------------------------------------------- |
| Backend process  | Fail                                      | Fail                                               |
| PostgreSQL       | Alive                                     | Readiness may fail; durable operations unavailable |
| Redis            | Alive                                     | Normally ready; edit locks/rate limits may degrade |
| RabbitMQ         | Alive                                     | Normally ready; Outbox accumulates                 |
| Outbox Publisher | Alive                                     | Ready; backlog grows                               |
| Audit Consumer   | Alive                                     | Ready; Activity/Audit delayed                      |
| MinIO            | Alive                                     | Non-object operations remain available             |
| Monitoring       | Alive                                     | Business processing continues                      |
| Keycloak         | Alive                                     | New authentication may fail safely                 |
| Vault            | Alive where current secrets remain usable | Depends on integration; no hard-coded fallback     |

Detailed dependency information shall not be exposed publicly.

# 7. Infrastructure Visibility
Initial monitoring covers:

* Linux host CPU, memory, disk, load, network, and uptime;
* Gateway availability, request failures, latency, and rate-limit events;
* PostgreSQL availability, connections, storage, and transaction state;
* RabbitMQ availability, queue depth, consumers, and DLQ;
* Redis availability and basic resource health;
* Keycloak, Vault, and MinIO availability and relevant failures.
  When PROD primary/replica is active, monitoring additionally covers replication state and lag.
  When Kubernetes is active, monitoring expands to nodes, Pods, Deployments, restarts, readiness, liveness, and resource usage.

# 8. Messaging and Outbox Visibility
RabbitMQ and Transactional Outbox are mandatory Version 1 observability targets.
Required visibility includes:

* broker availability, queue depth, and active consumers;
* Outbox pending count, oldest pending age, and publication failures;
* consumer failures, retries, and DLQ messages.
  A growing Outbox backlog or increasing oldest-event age requires investigation.

# 9. Audit, Security, and Technical Logs
Business Audit answers `Who changed business state?`, is stored in PostgreSQL, and is asynchronously materialized.
Technical logs answer `Why did the system behave this way?` and are centralized in Loki.
Security events remain a separate category defined by `security-architecture.md`.
A shared `correlationId` may connect these records without merging their responsibilities.

# 10. Sensitive Data
Technical logs shall not intentionally contain passwords, access/refresh tokens, Authorization headers, Vault tokens, database/RabbitMQ passwords, MinIO secret keys, or private keys.
Request/response payloads are not logged by default.
Future AI prompts and uploaded user content are not logged by default.

# 11. Dashboards
Initial dashboards:

1. Infrastructure Overview.
2. ProjectBoard Application / Gateway.
3. PostgreSQL.
4. RabbitMQ / Outbox.
5. Security / Operational Logs.
   Additional dashboards are introduced only when justified.
   Dashboard definitions should become version-controlled and reproducible.

# 12. Alerts
Initial actionable alerts include:

* Gateway or Backend unavailable;
* PostgreSQL unavailable;
* low disk space;
* RabbitMQ unavailable;
* sustained Outbox backlog growth;
* DLQ contains messages;
* backup failure or stale backup.
  Later alerts may include PostgreSQL replication lag and Kubernetes failures.
  Exact thresholds, windows, severity, grouping, and receivers remain open.
  Transient self-recovering anomalies should not create unnecessary critical alerts.

# 13. Environment and Build Identification
Metrics, logs, dashboards, and alerts shall identify `dev`, `stage`, or `prod` where applicable.
Telemetry should identify the originating service/component.
Running application instances should expose or log application version and Git commit for deployment traceability.

# 14. Backup Observability
Backup monitoring shall expose last successful backup, failure, backup age, and restore-test status.
Detailed backup implementation belongs to `backup-recovery.md`.

# 15. Failure Isolation
If Prometheus, Loki, Grafana, Alloy, or Alertmanager fails, normal ProjectBoard business processing continues according to actual business dependencies.
Collector retries or buffering must remain bounded so observability failure cannot exhaust host resources.
The observability stack shall expose its own health, including scrape failures, forwarding errors, storage pressure, and collection gaps.

# 16. Future OpenTelemetry and AI
Distributed tracing through OpenTelemetry is outside Version 1.

```text
Metrics + Logs + Correlation IDs → OpenTelemetry later
```

Future AI observability may include job counts, failures, queue time, processing duration, inference duration, and Ollama availability.
AI prompts and uploaded content remain excluded from technical telemetry by default.

# 17. Fixed Rules
1. Observability is part of Version 1.
2. Prometheus primarily pulls and stores operational metrics.
3. Loki stores centralized technical logs.
4. Source-side Alloy forwards local logs where required.
5. Grafana provides visualization and log exploration.
6. Alertmanager routes operational alerts.
7. Business Audit remains separate from technical logs.
8. Relevant operations propagate correlation IDs.
9. RabbitMQ, Outbox, and Activity/Audit Consumer are observable.
10. Telemetry identifies environment and component.
11. High-cardinality identifiers are not metric labels.
12. Secrets and authentication tokens are excluded from logs.
13. Monitoring failure does not stop business operations.
14. Metrics and log export remain outside the synchronous request path.
15. PostgreSQL replication is monitored when activated.
16. Backup failures and restore-test status are observable.
17. Alerts must remain actionable.
18. Monitoring access does not imply unrestricted administration.
19. Collector buffering is bounded.
20. OpenTelemetry is future work.

# 18. Open Decisions
1. Prometheus scrape intervals and retention.
2. Loki retention and Monitoring VM storage limits.
3. Alert thresholds, evaluation windows, severity, and receivers.
4. Grafana and monitoring-component authentication/authorization.
5. Source-side Alloy placement and whether central Alloy is also required.
6. Alloy buffering, retry, and drop behavior.
7. Telemetry TLS and authentication.
8. Exporter versions and packaging.
9. Time synchronization and clock-drift monitoring.
10. Keycloak and Vault health details.
11. Security-event retention.
12. Future Kubernetes telemetry topology.
13. Future OpenTelemetry topology, sampling, and retention.