# Software Requirements Specification
## ProjectBoard DevSecOps Lab

| Field | Value |
| --- | --- |
| Status | Working requirements baseline |
| Scope | Version 1 |
| Architecture | [High-Level Design](hld.md) |
| API Contract | `api/openapi.yaml` |
| Primary UI Language | English |
| Last updated | 2026-09-03 |

This document defines what ProjectBoard Version 1 shall provide.
Architecture and implementation details belong to the corresponding design documents.

# 1. Purpose and Scope
ProjectBoard is a web-based project-management application for small software-development teams.
It combines Backlog, Sprint Planning, and a Kanban Board.
The project principle is:

```text
Simple Business Logic
+
Progressively Complex Infrastructure
````

ProjectBoard is also a DevOps and DevSecOps learning project.

## 1.1 Version 1 Business Model

```text
Project
├── Members
├── Backlog
│   └── User Stories
│       └── Tasks
├── Sprints
│   └── selected User Stories
└── Kanban Board
    └── Board Columns
        └── User Stories
```

Scope rules:

* Backlog is a logical view, not a persistent entity.
* UserStory is the Kanban card; no separate `Card` entity exists.
* Task belongs to exactly one UserStory.
* Sprint planning occurs at UserStory level.
* Tasks are not independently selected into Sprints.
* Task assignee is outside Version 1.
* `Current Sprint` is outside Version 1.

# 2. Users and Roles

Version 1 uses `PROJECT_OWNER` and `MEMBER`.
There is no global ProjectBoard `ADMIN` role.
Authentication is delegated to Keycloak.
ProjectBoard shall not store user passwords.

## 2.1 Project Owner

The Project creator becomes `PROJECT_OWNER`.
The Project Owner may view, update, and delete the Project; add/remove Members; and perform all Member operations.
Ownership transfer is outside Version 1.

## 2.2 Member

A `MEMBER` may work with User Stories, Tasks, Sprints, Board Columns, and Kanban workflow.
A Member may not delete the Project, manage membership, or change Project ownership.

## 2.3 Project Isolation

Project data shall only be accessible to authenticated users who belong to the Project.
Authorization shall be enforced by the Backend.
Frontend restrictions are not sufficient authorization.

# 3. Functional Requirements

## 3.1 Authentication

* **FR-AUTH-01 — Login:** A registered user shall be able to authenticate through Keycloak.
* **FR-AUTH-02 — Registration:** Registration may be provided through Keycloak.
* **FR-AUTH-03 — Logout:** An authenticated user shall be able to terminate their authenticated session.
* **FR-AUTH-04 — Protected Access:** Protected functionality shall reject unauthenticated requests.

## 3.2 Projects

* **FR-PROJECT-01 — Create Project:** An authenticated user shall be able to create a Project with a name and optional description; the creator becomes `PROJECT_OWNER`.
* **FR-PROJECT-02 — List Projects:** A user shall only see Projects in which they participate.
* **FR-PROJECT-03 — View Project:** A Project member shall be able to view authorized Project data.
* **FR-PROJECT-04 — Update Project:** Only the Project Owner shall update Project-level information.
* **FR-PROJECT-05 — Delete Project:** Only the Project Owner shall hard-delete a Project; explicit confirmation is required.

## 3.3 Membership

* **FR-MEMBER-01 — View Available Users:** The Project Owner shall be able to view registered users available for membership.
* **FR-MEMBER-02 — Add Member:** The Project Owner shall be able to add a registered user as `MEMBER`.
* **FR-MEMBER-03 — Remove Member:** The Project Owner shall be able to remove a Member; removed users lose protected Project access.
* **FR-MEMBER-04 — Unique Membership:** The same external identity shall not have duplicate membership in one Project.

## 3.4 Backlog

* **FR-BACKLOG-01 — Backlog View:** Every Project shall provide a logical Backlog view.
* **FR-BACKLOG-02 — View Backlog:** Project members shall be able to view Project User Stories in the Backlog.
* **FR-BACKLOG-03 — Sprint Planning:** Project members shall be able to select Project User Stories for a Sprint.

## 3.5 User Stories

* **FR-STORY-01 — Create:** A Project member shall be able to create a User Story with a title and optional description.
* **FR-STORY-02 — View:** A Project member shall be able to view a User Story belonging to their Project.
* **FR-STORY-03 — Update:** A Project member shall be able to update a User Story.
* **FR-STORY-04 — Delete:** A Project member shall be able to hard-delete a User Story; confirmation is required when dependent data are affected.
* **FR-STORY-05 — Ownership:** Every User Story shall belong to exactly one Project.

## 3.6 Tasks

* **FR-TASK-01 — Create:** A Project member shall be able to create a Task inside a User Story with title, optional description, and completion state.
* **FR-TASK-02 — Update:** A Project member shall be able to update a Task.
* **FR-TASK-03 — Delete:** A Project member shall be able to hard-delete a Task.
* **FR-TASK-04 — Complete:** A Project member shall be able to mark a Task completed or incomplete again.
* **FR-TASK-05 — Ownership:** Every Task shall belong to exactly one User Story.
* **FR-TASK-06 — Sprint Relationship:** Tasks shall not be independently selected into Sprints; Sprint association occurs through UserStory.

## 3.7 Sprints

* **FR-SPRINT-01 — Create:** A Project member shall be able to create a Sprint with name, start date, and end date.
* **FR-SPRINT-02 — Dates:** Sprint end date shall not precede Sprint start date.
* **FR-SPRINT-03 — Update:** A Project member shall be able to update Sprint information.
* **FR-SPRINT-04 — Delete:** A Project member shall be able to delete a Sprint.
* **FR-SPRINT-05 — Add Story:** A Project member shall be able to associate a Project User Story with a Sprint.
* **FR-SPRINT-06 — Remove Story:** A Project member shall be able to remove a User Story from a Sprint without deleting it.
* **FR-SPRINT-07 — No Current Sprint:** Version 1 shall not require a dedicated `Current Sprint` concept.
* **FR-SPRINT-08 — No Automatic Scheduling:** Sprint dates shall not automatically start or complete a Sprint.

## 3.8 Kanban Board

* **FR-BOARD-01 — Project Board:** Each Project shall provide a Kanban Board view.
* **FR-BOARD-02 — UserStory as Card:** Each Kanban card shall represent a UserStory; no separate `Card` entity exists.
* **FR-BOARD-03 — Board Content:** Project members shall be able to view and organize Project User Stories on the Board.

## 3.9 Board Columns

* **FR-COLUMN-01 — Defaults:** A new Project shall initially provide `To Do`, `In Progress`, and `Done`.
* **FR-COLUMN-02 — Create:** Project members shall be able to create Board Columns.
* **FR-COLUMN-03 — Rename:** Project members shall be able to rename Board Columns.
* **FR-COLUMN-04 — Delete:** An empty Board Column may be deleted; Stories must be moved first.
* **FR-COLUMN-05 — Move Story:** Project members shall move User Stories between Columns and persist placement.
* **FR-COLUMN-06 — Order:** Board Columns shall have a persistent workflow order.

## 3.10 Concurrent Editing

* **FR-CONC-01 — Edit Lock:** UserStory editing shall support a temporary Redis edit lock.
* **FR-CONC-02 — TTL:** The edit lock shall have a finite TTL and expire automatically.
* **FR-CONC-03 — Renewal:** The lock shall be renewable while editing continues.
* **FR-CONC-04 — Release:** The application should release the lock on Save or Cancel where possible.
* **FR-CONC-05 — Advisory:** Redis edit locks shall not be the authoritative data-integrity mechanism.
* **FR-CONC-06 — Optimistic Locking:** UserStory updates shall use optimistic locking and reject stale writes.

## 3.11 Messaging

* **FR-MSG-01 — Outbox:** Business changes and corresponding asynchronous Outbox events shall be persisted atomically.
* **FR-MSG-02 — RabbitMQ:** RabbitMQ shall provide asynchronous event transport.
* **FR-MSG-03 — Delivery:** Messaging shall use at-least-once delivery.
* **FR-MSG-04 — Idempotency:** Consumers shall safely process duplicate events.
* **FR-MSG-05 — Retry:** Temporary consumer failures shall use bounded retry.
* **FR-MSG-06 — DLQ:** Repeatedly failing messages shall move to a Dead Letter Queue.
* **FR-MSG-07 — Identity:** Each event shall have a unique `eventId`; relevant operations shall propagate `correlationId`.
* **FR-MSG-08 — Audit Consumer:** The initial asynchronous consumer shall create Project Activity/Audit records.
* **FR-MSG-09 — Eventual Consistency:** Business state may commit before corresponding Activity/Audit is created.
* **FR-MSG-10 — Delete Events:** Hard-delete events shall contain enough information for required historical Activity/Audit.

## 3.12 Activity/Audit

* **FR-AUDIT-01 — History:** Project members shall be able to view relevant Project Activity/Audit information.
* **FR-AUDIT-02 — Append-Only:** Normal business APIs shall not modify historical Activity/Audit records.
* **FR-AUDIT-03 — Separation:** Business Activity/Audit shall remain separate from technical and security logs.

## 3.13 REST API

* **FR-API-01 — REST:** Core ProjectBoard functionality shall be exposed through REST.
* **FR-API-02 — Versioning:** Version 1 API routes shall use `/api/v1/`.
* **FR-API-03 — OpenAPI:** The contract shall be maintained in `api/openapi.yaml` before corresponding Frontend/Backend implementation.
* **FR-API-04 — Errors:** The API shall return consistent errors for validation, authentication, authorization, not found, optimistic conflicts, business violations, and internal errors.
* **FR-API-05 — Validation:** Incoming data shall be validated by the Backend.

# 4. Non-Functional Requirements

## 4.1 Security

* **NFR-SEC-01:** Protected functionality shall require Keycloak authentication.
* **NFR-SEC-02:** Business authorization shall be enforced by the Backend.
* **NFR-SEC-03:** Users shall not access Projects to which they do not belong.
* **NFR-SEC-04:** ProjectBoard shall not store user passwords.
* **NFR-SEC-05:** Secrets shall remain outside source code, Git, application images, and browser code.
* **NFR-SEC-06:** Angular shall not contain a confidential client secret.
* **NFR-SEC-07:** Public application and authentication traffic shall use HTTPS.
* **NFR-SEC-08:** Internal infrastructure services shall not be publicly exposed.
* **NFR-SEC-09:** Version 1 shall provide basic rate limiting for selected traffic.
* **NFR-SEC-10:** Security-relevant events shall be observable.

## 4.2 Data Integrity

* **NFR-DATA-01:** Authoritative business data shall survive normal application/container restart.
* **NFR-DATA-02:** Database schema evolution shall use Flyway.
* **NFR-DATA-03:** Invalid cross-Project and duplicate relationships shall be prevented.
* **NFR-DATA-04:** Related business changes shall be transactional where partial persistence violates invariants.
* **NFR-DATA-05:** UserStory stale writes shall be detected.
* **NFR-DATA-06:** Version 1 uses hard deletion; soft delete is outside Version 1.

## 4.3 Maintainability and Testability

* **NFR-MAIN-01:** Frontend, Backend, persistence, infrastructure, and API concerns shall remain separated.
* **NFR-MAIN-02:** The Backend shall use a Spring Boot Modular Monolith.
* **NFR-MAIN-03:** Core business logic should not depend unnecessarily on specific infrastructure implementations.
* **NFR-TEST-01:** Important business rules shall have automated tests.
* **NFR-TEST-02:** Authorization-sensitive behavior shall have automated tests.
* **NFR-TEST-03:** Important REST endpoints shall be independently testable.
* **NFR-TEST-04:** Important concurrency and messaging behavior shall be testable.

## 4.4 Runtime and Networking

* **NFR-RUNTIME-01:** Frontend and Backend shall be containerizable.
* **NFR-RUNTIME-02:** DEV shall use Docker Compose.
* **NFR-RUNTIME-03:** STAGE shall use Docker Compose.
* **NFR-RUNTIME-04:** The accepted PROD target is a three-node Kubernetes cluster.
* **NFR-RUNTIME-05:** Production PostgreSQL shall remain outside Kubernetes.
* **NFR-RUNTIME-06:** Containers should run with minimum required privileges.
* **NFR-NET-01:** Public application traffic shall enter through the Nginx Gateway.
* **NFR-NET-02:** DEV, STAGE, and PROD shall use isolated data, credentials, and supporting state.
* **NFR-NET-03:** Lower environments shall not access PROD data or secrets.

## 4.5 CI/CD and DevSecOps

* **NFR-CI-01:** Pull Requests shall trigger automated validation.
* **NFR-CI-02:** CI shall build Frontend and Backend and execute mandatory tests.
* **NFR-CI-03:** Security checks shall progressively include dependency, SAST, secret, and container-image scanning.
* **NFR-CI-04:** Failed mandatory checks shall block readiness for merge.
* **NFR-CI-05:** Application images shall be built once and promoted unchanged.
* **NFR-CI-06:** Promoted artifacts shall have an immutable identifier.
* **NFR-CI-07:** PROD deployment shall require explicit approval.

## 4.6 Observability

* **NFR-OBS-01:** Important application and infrastructure metrics shall be available.
* **NFR-OBS-02:** Technical logs shall be centrally collectable and searchable.
* **NFR-OBS-03:** RabbitMQ, Outbox, and DLQ health shall be observable.
* **NFR-OBS-04:** The platform shall support actionable alerts.
* **NFR-OBS-05:** Relevant operations shall propagate correlation identifiers.
* **NFR-OBS-06:** Monitoring failure shall not block business processing.
* **NFR-OBS-07:** Secrets and authentication tokens shall not intentionally appear in logs.

## 4.7 Messaging Reliability

* **NFR-MSG-01:** RabbitMQ availability shall not be required when publication intent is durably persisted in the Outbox.
* **NFR-MSG-02:** Consumers shall support at-least-once delivery.
* **NFR-MSG-03:** Duplicate messages shall not create duplicate logical effects.
* **NFR-MSG-04:** Repeated failures shall remain available for controlled investigation.
* **NFR-MSG-05:** DEV, STAGE, and PROD messaging state shall remain isolated.

## 4.8 Backup and Recovery

* **NFR-BACKUP-01:** Required persistent state shall have scheduled backups.
* **NFR-BACKUP-02:** Backup data shall be stored outside the source being protected.
* **NFR-BACKUP-03:** Backup creation shall be verifiable.
* **NFR-BACKUP-04:** Restore procedures shall be documented.
* **NFR-BACKUP-05:** A backup shall not be considered complete until restoration has been tested.
* **NFR-BACKUP-06:** Restore tests should not modify active business data.

## 4.9 Infrastructure as Code

* **NFR-IAC-01:** Terraform shall manage VM infrastructure.
* **NFR-IAC-02:** Ansible shall configure operating systems and server roles.
* **NFR-IAC-03:** Manual infrastructure configuration shall progressively become version-controlled automation.

## 4.10 Performance, Usability, and Learning

* **NFR-PERF-01:** Version 1 shall support expected small-team usage without unnecessary blocking.
* **NFR-PERF-02:** Asynchronous processing shall not unnecessarily block REST operations.
* **NFR-UX-01:** The UI shall prioritize clarity.
* **NFR-UX-02:** English shall be the primary Version 1 interface language.
* **NFR-EDU-01:** The developer shall understand why each major technology is used and how to verify it.
* **NFR-EDU-02:** Technologies shall not be introduced solely to increase tool count.

# 5. Out of Scope

## 5.1 Product Features

* Retrospectives.
* Notifications.
* Chat.
* Realtime board synchronization.
* WebSockets.
* General attachments.
* Native mobile applications.

## 5.2 Business Model

* Global `ADMIN` / `SUPER_ADMIN`.
* Ownership transfer.
* Custom roles and advanced permissions.
* Task assignee.
* `Current Sprint`.
* Task/UserStory dependencies and subtasks.
* Epics, roadmaps, milestones, labels, custom fields, and time tracking.

## 5.3 Advanced Sprint Features

* Automatic Sprint start/completion.
* Story points, velocity, burndown charts, capacity planning, and advanced Sprint analytics.

## 5.4 AI

Version 1 excludes active LLM/Vision integration, screenshot analysis, AI-generated Tasks, and autonomous AI business-data changes.

## 5.5 High Availability

Version 1 does not require automatic PostgreSQL failover, Gateway HA, zero-downtime operation, or multi-region deployment.

# 6. Version 1 Capability Summary

| Area           | Required Capability               |
| -------------- | --------------------------------- |
| Authentication | Keycloak/OIDC                     |
| Authorization  | Backend Project roles             |
| Projects       | CRUD with owner restrictions      |
| Membership     | Owner + Members                   |
| User Stories   | CRUD + optimistic locking         |
| Tasks          | CRUD + completion                 |
| Backlog        | Logical view                      |
| Sprints        | CRUD + UserStory selection        |
| Kanban         | Columns + UserStory movement      |
| Concurrency    | Redis edit lock                   |
| API            | REST `/api/v1` + OpenAPI          |
| Persistence    | PostgreSQL + Flyway               |
| Messaging      | RabbitMQ + Transactional Outbox   |
| Reliability    | retry + idempotency + DLQ         |
| Audit          | asynchronous Activity/Audit       |
| Security       | HTTPS + isolation + rate limiting |
| DEV/STAGE      | Docker Compose                    |
| Observability  | metrics + logs + alerts           |
| Backup         | scheduled backup + restore test   |
| CI/CD          | build + test + promote            |
| IaC            | Terraform + Ansible               |
| PROD target    | Kubernetes + external PostgreSQL  |

# 7. Delivery Priority

## Must

All Version 1 requirements in Sections 2–4 are required.

## Should

* Full Vault runtime integration.
* Container hardening and security scanning.
* Richer alerts and administrative controls.
* Stronger internal TLS where justified.

## Could

* Additional UI languages and MinIO-backed user features.
* Advanced rate limiting and richer health diagnostics.
* Kubernetes activation, NetworkPolicies, RBAC hardening, GitOps, and Argo CD.
* PostgreSQL failover experiments.

## Won't in Version 1

Items listed in Section 5 are excluded.

# 8. Definition of Done

## 8.1 Feature

A feature is Done when:

* implementation matches a defined requirement;
* OpenAPI is synchronized where relevant;
* authorization and Project isolation are correct;
* required tests pass and CI is green;
* secrets are not committed;
* relevant documentation is current;
* the developer can explain the implementation.

## 8.2 Infrastructure Phase

An infrastructure phase is Done when:

* the capability works;
* configuration is reproducible;
* failure behavior is understood;
* relevant tests pass;
* correct operation can be demonstrated.

## 8.3 Version 1

Version 1 is complete when:

1. all Version 1 requirements are implemented;
2. authentication, authorization, and Project isolation work;
3. core Project, Membership, UserStory, Task, Sprint, and Kanban workflows work;
4. Task assignee and Current Sprint are absent;
5. Redis edit locking and UserStory optimistic locking work;
6. PostgreSQL persistence and Flyway work;
7. RabbitMQ, Outbox, retry, idempotency, and DLQ work;
8. Activity/Audit is generated asynchronously;
9. OpenAPI matches the implemented REST API;
10. required tests and CI checks pass;
11. secrets are not embedded in source, images, or browser code;
12. DEV and STAGE are reproducible and isolated;
13. required metrics, logs, and alerts are available;
14. backup and isolated restore testing work;
15. infrastructure and documentation are reproducible and current;
16. the developer can explain and demonstrate the system.
