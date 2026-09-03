# Application Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Domain | [Domain Model](domain-model.md) |
| API | `api/openapi.yaml` |
| Last updated | 2026-09-03 |
This document defines how ProjectBoard use cases flow through Frontend, Backend modules, and infrastructure boundaries.
Database schema, RabbitMQ topology, deployment, and infrastructure configuration belong to dedicated designs.
# 1. Application Structure
```text
Angular Frontend
      ↓
REST API
      ↓
Spring Boot Backend
      ↓
Application Layer
      ↓
Domain Modules
      ↓
Repositories / Infrastructure
```
The Backend is the authoritative boundary for business logic and authorization.
# 2. Backend Layers
```text
API / Controller
      ↓
Application Service / Use Case
      ↓
Domain
      ↓
Repository
```
Rules:
1. Controllers handle HTTP concerns only.
2. Application services coordinate use cases and module interaction.
3. Domain code owns business rules and invariants.
4. Repositories belong to their owning modules.
5. Infrastructure adapters implement technical integrations.
# 3. Modular Monolith
Initial modules:
```text
project
membership
story
task
sprint
board
shared
```
Each module owns its application logic, domain rules, and persistence boundary.
Modules communicate through explicit application interfaces.
A module shall not directly manipulate another module's repository.
`shared` contains technical primitives only and owns no business concepts.
# 4. Cross-Module Use Cases
The application layer coordinates operations involving multiple modules.
## 4.1 Create Project
```text
Create Project
→ Project Module
→ Membership Module
→ Board Module
→ Outbox
→ Commit
```
The operation creates the Project, creator `PROJECT_OWNER` membership, default BoardColumns, and required Outbox event in one transaction.
## 4.2 Add UserStory to Sprint
```text
Request
→ Resolve Sprint
→ Resolve UserStory
→ Validate same Project
→ Associate
```
A Sprint may select only UserStories from its own Project.
## 4.3 Move UserStory
```text
Request
→ Resolve UserStory
→ Resolve target BoardColumn
→ Validate same Project
→ Update placement/order
```
The operation must preserve Project authorization and optimistic locking.
# 5. Authorization Flow
```text
HTTP Request
→ Validate Keycloak token
→ Resolve identity
→ Resolve owning Project
→ Check membership and role
→ Execute use case
```
Rules:
- authentication does not imply Project access;
- resource IDs alone never grant authorization;
- Backend authorization is authoritative;
- Frontend visibility is not a security boundary;
- ambiguous ownership or authorization fails closed.
# 6. Transaction Boundaries
One PostgreSQL transaction is used when partial persistence would violate an invariant.
Examples:
- Project + owner membership + default BoardColumns;
- business mutation + Outbox event;
- cross-module operations requiring immediate consistency.
Asynchronous consumers run outside the original business transaction.
Detailed persistence rules belong to `database-design.md`.
# 7. Messaging Integration
Core business use cases remain synchronous REST operations.
```text
Application Use Case
→ Business Change + Outbox Event
→ COMMIT
→ Outbox Publisher
→ RabbitMQ
```
In Version 1, the Outbox Publisher and Activity/Audit Consumer run inside the Backend process.
Delivery semantics belong to `messaging-design.md`.
# 8. Concurrency
UserStory editing combines temporary Redis edit locks for collaboration UX with PostgreSQL optimistic locking for durable integrity.
Redis locks do not authorize users and do not replace optimistic locking.
Exact polling and lock-renewal behavior remains implementation detail.
# 9. API Contract
REST routes use `/api/v1/`.
The canonical contract is `api/openapi.yaml`.
OpenAPI is defined before corresponding Frontend and Backend implementation.
Controllers and Frontend clients shall follow the same contract.
# 10. Error Handling
The API distinguishes:
- validation errors;
- unauthenticated requests;
- forbidden operations;
- missing resources;
- optimistic-lock conflicts;
- business-rule violations;
- internal errors.
The exact JSON error schema belongs to OpenAPI.
Internal exceptions shall not expose implementation details.
# 11. Frontend Structure
Conceptual Angular structure:
```text
app
├── core
├── auth
├── projects
├── stories
├── tasks
├── sprints
├── board
└── shared
```
The Frontend:
- uses the Backend REST API;
- handles navigation and presentation;
- may hide unavailable actions for UX;
- never replaces Backend authorization;
- never directly accesses PostgreSQL, Redis, RabbitMQ, Vault, MinIO, or AI services.
Exact Angular state management remains open.
# 12. DTO and Mapping Boundary
API DTOs are not domain entities.
Controllers exchange API request/response models with the application layer.
Mapping prevents transport concerns from leaking into domain code.
Manual mapping or a mapping library may be used.
# 13. Validation
Validation occurs at three levels:
- request validation at the API boundary;
- business validation in application/domain logic;
- structural integrity in PostgreSQL.
Frontend validation improves UX but is not authoritative.
# 14. Fixed Rules
1. Backend is a Spring Boot Modular Monolith.
2. Controllers contain no business logic.
3. Application services coordinate use cases.
4. Modules own their repositories and domain rules.
5. Cross-module repository manipulation is prohibited.
6. `shared` owns no business domain.
7. Backend enforces Project authorization.
8. One transaction may span modules when immediate consistency requires it.
9. Business mutation and required Outbox event are atomic.
10. Frontend uses only approved Backend APIs.
11. OpenAPI is the canonical HTTP contract.
12. API DTOs remain separate from domain entities.
13. PostgreSQL optimistic locking is authoritative for stale writes.
14. Infrastructure failures must not bypass validation or authorization.
# 15. Open Decisions
1. Java package structure inside modules.
2. Application-interface pattern between modules.
3. DTO mapping approach.
4. Global API error schema.
5. Angular state-management approach.
6. Generated versus handwritten OpenAPI client.
7. Transaction annotation placement.
8. HTTP polling strategy for Redis edit locks.