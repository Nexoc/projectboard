# High-Level Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Scope | Version 1 baseline and target evolution |
| Requirements | [SRS](srs.md) |
| Last updated | 2026-09-03 |
This document is the architectural entry point for ProjectBoard. Detailed behavior belongs to the specialized design documents.

# 1. Purpose
ProjectBoard is a small Kanban/project-management application and a DevOps/DevSecOps learning platform.
Core principle: `Simple Business Logic + Progressively Complex Infrastructure`.
The architecture prioritizes understandable business logic, explicit boundaries, reproducibility, security, observability, and recoverability.

# 2. Architecture Goals
1. Simple business domain and Modular Monolith Backend.
2. Server-side authentication and Project authorization.
3. Authoritative state outside disposable application containers.
4. Immutable environment-independent artifacts.
5. Isolation of DEV, STAGE, and PROD.
6. Progressive Infrastructure as Code and automation.
7. Centralized observability and tested recovery.
8. Evolution from Docker Compose to Kubernetes.
9. Future AI remains outside the core application.
High availability and zero-downtime operation are not Version 1 requirements.

# 3. System Context
Main actors: ProjectBoard User and Developer / Infrastructure Administrator.
Project roles are `PROJECT_OWNER` and `MEMBER`; there is no global ProjectBoard `ADMIN`.

| System | Responsibility |
| --- | --- |
| Keycloak | Authentication |
| GitHub / Actions | Source, review, CI/CD |
| GHCR | Container registry |
| Vault | Secrets |
| MinIO | Object storage |
| Monitoring Platform | Metrics, logs, dashboards, alerts |
| Future AI Service | AI boundary |
| Ollama / Vision | Future local inference |

Technical administration does not automatically grant ProjectBoard business permissions.
See `diagrams/system-context.md`.

# 4. Application Architecture
The Spring Boot Backend uses a Modular Monolith with modules:
`project`, `membership`, `story`, `task`, `sprint`, `board`, `shared`.

Modules own their domain rules and persistence boundaries and communicate through explicit application interfaces.
Microservices are not required for Version 1.

| Component | Responsibility |
| --- | --- |
| Angular Frontend | Browser UI |
| Spring Boot Backend | REST API, authorization, business logic |
| PostgreSQL | Authoritative persistence |
| Redis | Temporary coordination |
| RabbitMQ | Async transport |
| Keycloak | Authentication |
| Nginx Gateway | Public HTTPS entry |
| Vault | Secrets |
| MinIO | Binary objects |
| Monitoring Platform | Observability |

All authoritative business changes pass through the Backend.
The Frontend never directly accesses internal data/services.

# 5. API Boundary
ProjectBoard uses REST under `/api/v1/`.
OpenAPI is defined before corresponding Frontend/Backend implementation.
Canonical contract: `api/openapi.yaml`.

# 6. Business and Data
`Project` is the business, authorization, and data-isolation boundary.
Core rules: UserStory is the Kanban card; no `Card` entity; Backlog is logical; Task belongs to UserStory; Sprint selects UserStories; no Task assignee or `Current Sprint`; UserStory uses optimistic locking; Version 1 uses hard delete.

| Data | Owner |
| --- | --- |
| Business state | PostgreSQL |
| Identity-provider state | Keycloak DB |
| Temporary state | Redis |
| Binary objects | MinIO |
| Async transport | RabbitMQ |

Flyway manages PostgreSQL schema evolution.
Detailed rules belong to `domain-model.md` and `database-design.md`.

# 7. Messaging and Concurrency
RabbitMQ and Transactional Outbox are mandatory in Version 1.
Flow: `Backend → PostgreSQL business change + Outbox → RabbitMQ → Activity/Audit Consumer`.
Delivery is at-least-once; consumers are idempotent; retry and DLQ are required.
RabbitMQ is transport, not business storage.
Outbox Publisher and Activity/Audit Consumer run inside the Backend process in Version 1.
UserStory editing combines Redis edit locks with PostgreSQL optimistic locking; PostgreSQL remains authoritative.

# 8. Security
Keycloak owns authentication; Backend owns Project authorization.
Public application and OIDC traffic enters through the Nginx Gateway over HTTPS.
Security follows default deny, least privilege, server-side authorization, environment isolation, and externalized secrets.
Internal data and administrative services are not publicly exposed.
Exact browser OIDC/token-storage behavior remains open.
See `security-architecture.md` and `network-design.md`.

# 9. Observability
Observability is part of Version 1.
The Monitoring VM hosts Prometheus, Grafana, Loki, Grafana Alloy, and Alertmanager.
Prometheus pulls metrics; source-side Alloy forwards logs to Loki.
RabbitMQ, Outbox, and Activity/Audit processing are observable.
Business Audit remains separate from technical logs.
Monitoring failure does not stop core business processing.

# 10. Deployment
| Environment | Model |
| --- | --- |
| DEV | Dedicated VM + Docker Compose |
| STAGE | Dedicated VM + Docker Compose |
| PROD | Three-node Kubernetes target |

Shared placement:
| Role | Placement |
| --- | --- |
| Nginx | `gateway` VM |
| Keycloak + Keycloak DB + Vault | `security` VM |
| MinIO | `storage` VM |
| Observability stack | `monitoring` VM |

PROD PostgreSQL runs outside Kubernetes as `prod-db-primary → prod-db-replica`.
The replica is not a backup; automatic failover is future work.
All Linux VMs use Debian 13.

# 11. Infrastructure as Code
Responsibilities: `Terraform → VM lifecycle`, `Ansible → OS/server configuration`, `Compose/Kubernetes → application runtime`.
Terraform outputs should support Ansible inventory.
The architecture is not tied to one VirtualBox Terraform provider.

# 12. CI/CD
Delivery: `Build/Test/Scan → GHCR → DEV → STAGE → PROD`.
Frontend and Backend images are built once and promoted unchanged.
Configuration and secrets remain external.
DEV becomes automatic after successful integration CI when deployment is active.
PROD requires explicit approval.
GitOps follows direct Kubernetes deployment; Argo CD runs inside Kubernetes.

# 13. Backup and Recovery
Version 1 requires scheduled backups, storage outside the protected source, verification, documented manual restore, and periodic isolated restore tests.
Protected scope includes ProjectBoard PostgreSQL, Keycloak DB, MinIO when active, and Vault when operational.
Redis is disposable; RabbitMQ is transport; replication does not replace backup.

# 14. Future Architecture
Future capabilities include GitOps/Argo CD, Gateway HA, PostgreSQL automatic failover, OpenTelemetry, and AI.
AI is outside Version 1 and follows `Backend → Outbox/RabbitMQ → AI Service → Ollama/Vision`.
AI has no direct ProjectBoard database mutation path and its output becomes business data only through Backend validation and explicit user acceptance.

# 15. Architectural Decisions
| ID | Decision | Status |
| --- | --- | --- |
| HLD-001 | Spring Boot Modular Monolith | Accepted |
| HLD-002 | Angular Frontend | Accepted |
| HLD-003 | PostgreSQL business source of truth | Accepted |
| HLD-004 | Keycloak authentication | Accepted |
| HLD-005 | Backend Project authorization | Accepted |
| HLD-006 | Flyway migrations | Accepted |
| HLD-007 | External configuration and secrets | Accepted |
| HLD-008 | Docker Compose for DEV/STAGE | Accepted |
| HLD-009 | PROD PostgreSQL outside Kubernetes | Accepted target |
| HLD-010 | RabbitMQ + Transactional Outbox in V1 | Accepted |
| HLD-011 | Redis temporary edit locks/state | Accepted |
| HLD-012 | Shared Security VM | Accepted |
| HLD-013 | Dedicated Storage VM / MinIO | Accepted |
| HLD-014 | Dedicated Monitoring VM | Accepted |
| HLD-015 | Three-node PROD Kubernetes | Accepted target |
| HLD-016 | GitOps / Argo CD after direct Kubernetes | Accepted future path |
| HLD-017 | AI inference on Windows GPU host | Accepted future constraint |
| HLD-018 | AI as separate service | Accepted future architecture |
| HLD-019 | Public traffic through Nginx Gateway | Accepted |
| HLD-020 | Async Activity/Audit through RabbitMQ | Accepted |

# 16. Constraints
Home-lab baseline: Windows host, VirtualBox, Debian 13 VMs, Ryzen 7 3700X, 96 GB RAM, RTX 3060 16 GB, 2 TB NVMe.
Version 1 does not require multi-region deployment, sharding, automatic failover, or zero-downtime deployment.
Resource allocations may change after measurement.

# 17. Open Decisions
1. VirtualBox Terraform provider and VM network/IP layout.
2. Management-access mechanism and internal TLS boundaries.
3. Keycloak realm/client topology and browser OIDC details.
4. Vault bootstrap, recovery, and workload authentication.
5. Kubernetes distribution, CNI, ingress, and namespaces.
6. Board ordering and Sprint-to-UserStory persistence.
7. Outbox claiming/retention and RabbitMQ topology.
8. Monitoring retention and alert thresholds.
9. Backup schedule, retention, destination, and future RPO/RTO.
10. Project phase when PROD Kubernetes becomes active.

# 18. Detailed Designs
`domain-model.md`, `application-design.md`, `database-design.md`, `messaging-design.md`, `security-architecture.md`, `network-design.md`, `observability-design.md`, `deployment-architecture.md`, `cicd-design.md`, `backup-recovery.md`, `ai-architecture.md`, and `diagrams/`.