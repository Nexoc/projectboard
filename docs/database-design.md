# ProjectBoard Database Design
| Field | Value |
| --- | --- |
| Status | Draft logical design |
| Parent | [High-Level Design](hld.md) |
| Domain | [Domain Model](domain-model.md) |
| ERD | [Conceptual ERD](diagrams/erd.md) |
| Last updated | 2026-09-03 |
This document defines ProjectBoard persistence, integrity, transactions, and schema evolution.

# 1. Storage Ownership
| Store | Responsibility |
| --- | --- |
| PostgreSQL | Business data, Activity/Audit, Outbox, deduplication |
| Keycloak DB | Identity-provider state |
| Redis | Temporary coordination state |
| MinIO | Binary objects when required |
| RabbitMQ | Message transport |
PostgreSQL is authoritative for ProjectBoard business data. External services shall not directly modify ProjectBoard business tables.

# 2. PostgreSQL Scope
ProjectBoard PostgreSQL persists Identity References, Projects, ProjectMembers, UserStories, Tasks, Sprints, BoardColumns, Board placement/order, Activity/Audit, Outbox, deduplication state, and object metadata when required.

# 3. Project Ownership
| Record | Ownership |
| --- | --- |
| ProjectMember | Project |
| UserStory | Project |
| Task | UserStory → Project |
| Sprint | Project |
| BoardColumn | Project |
| Activity/Audit | retained Project ID |
| Outbox | retained Project/event context |
| Object metadata | Project |
Cross-Project Story/Column and Sprint/Story associations are invalid. Backend authorization remains mandatory.

# 4. Identity References
Keycloak owns authentication data. ProjectBoard stores only the local identity reference required for business relationships.
```text
Keycloak Identity → Identity Reference → ProjectMember
```
Membership is unique for `Project + Identity Reference`. ProjectBoard stores no passwords or Keycloak credentials. Identity Reference lifecycle after Keycloak identity removal remains open.

# 5. Relational Integrity
PostgreSQL uses primary keys, foreign keys, non-null, unique, and check constraints where appropriate.
Required rules:
1. one membership per identity per Project;
2. role limited to `PROJECT_OWNER` or `MEMBER`;
3. Task requires UserStory;
4. UserStory, Sprint, and BoardColumn require Project;
5. cross-Project associations are invalid;
6. optimistic-lock versions remain valid.

# 6. Transactions
Related durable changes use one transaction when partial persistence would violate an invariant.
Examples include Project creation with owner/default columns, membership changes, Story updates, Sprint associations, Board movement, hard deletion, and business mutation + Outbox event.
```text
BEGIN
Business changes
Outbox event
COMMIT
```

# 7. Transactional Outbox
Business mutation and Outbox event are persisted atomically.
```text
PostgreSQL transaction
├── Business change
└── Outbox event
        ↓
      COMMIT
```
RabbitMQ availability is not required during the transaction. Publication behavior belongs to `messaging-design.md`.
Conceptual Outbox data: `eventId`, `eventType`, `aggregateType`, `aggregateId`, `projectId`, `correlationId`, `createdAt`, `payload`, publication state.
Exact schema, claiming, cleanup, and retention remain open.

# 8. Idempotency and Audit
At-least-once messaging may deliver duplicates. Persistent deduplication may record processed `eventId` values.
Business Activity/Audit is stored in PostgreSQL, append-only through normal APIs, asynchronous, and separate from technical/security logs.
Required audit history may survive source-entity hard deletion.

# 9. Optimistic Locking
UserStory uses optimistic locking.
```text
UserStory
├── id
└── version
```
Stale writes are rejected when the expected version differs from the stored version. Redis edit locks do not replace database concurrency control. Additional optimistic locking remains open.

# 10. Hard Delete
Version 1 uses hard delete. Soft delete and undelete are outside Version 1.
Dependent relationships explicitly use suitable `CASCADE`, `RESTRICT`, or `SET NULL` behavior. Deletion must not affect unrelated Project data. Exact FK policies remain open.

# 11. Board Persistence
BoardColumns are persistent Project-owned records. New Projects contain `To Do`, `In Progress`, and `Done`.
Persisted Board state includes BoardColumn order, UserStory column placement, and UserStory order.
A Story may reference only a Column from the same Project. No persistent `Board` entity is required. Exact ordering representation remains open.

# 12. Sprint Persistence
Sprint belongs to one Project and selects UserStories from the same Project. Tasks are not independently associated with Sprints.
The physical Sprint/UserStory association remains open; an association table is allowed but not mandated.

# 13. MinIO Metadata
When binary-object functionality exists, PostgreSQL owns business metadata/authorization context and MinIO owns binary bytes.
PostgreSQL and MinIO do not share one ACID transaction. Reconciliation is defined when a concrete object-storage feature is implemented.

# 14. Flyway
All ProjectBoard PostgreSQL schema changes use Flyway.
Rules:
1. migrations are version-controlled;
2. applied versioned migrations are not modified;
3. corrections use new migrations;
4. DEV, STAGE, and PROD use the same migration order;
5. migration failure stops deployment;
6. manual production DDL is exceptional.
Potentially destructive changes should use `Expand → Migrate → Contract` where needed.

# 15. Environment Separation
DEV, STAGE, and PROD use separate databases and credentials.
```text
DEV   → ProjectBoard DB
STAGE → ProjectBoard DB
PROD  → ProjectBoard DB
```
Cross-environment business database access is prohibited.

# 16. Production PostgreSQL
Production PostgreSQL runs outside Kubernetes.
```text
prod-db-primary
       ↓ replication
prod-db-replica
```
The primary is the normal application endpoint. The replica supports replication/recovery learning. Automatic failover is outside Version 1. Replication does not replace backup.

# 17. Database Privileges
Application runtime shall not use PostgreSQL superuser privileges.
Target identities: Runtime, Migration, and Administrative. Each follows least privilege.
Whether runtime and migration identities must be separate immediately remains open.

# 18. Consistency Model
| Boundary | Consistency |
| --- | --- |
| Related business rows | PostgreSQL transaction |
| Business state + Outbox | PostgreSQL transaction |
| RabbitMQ processing | Eventual, at-least-once |
| Activity/Audit | Eventual |
| PostgreSQL + Redis | PostgreSQL authoritative |
| PostgreSQL + MinIO | Coordinated eventual |
| Keycloak + Identity Reference | Keycloak authoritative |
| Primary + replica | PostgreSQL replication |

# 19. Backup Boundary
Recovery details belong to `backup-recovery.md`.
ProjectBoard PostgreSQL and Keycloak DB require backup; MinIO requires backup when active; Redis is disposable; RabbitMQ is transport; replica is not backup; Outbox protects unpublished event intent.

# 20. Fixed Rules
1. PostgreSQL is the business source of truth.
2. Keycloak owns authentication data.
3. DEV, STAGE, and PROD databases are separate.
4. Protected records have traceable Project ownership.
5. Database constraints protect structural integrity where practical.
6. Backend authorization remains mandatory.
7. Related durable changes use transactions.
8. Business mutation and Outbox creation are atomic.
9. Flyway manages schema evolution.
10. RabbitMQ is transport only.
11. Consumers tolerate duplicate delivery.
12. Redis contains no irreplaceable business data.
13. UserStory uses optimistic locking.
14. Version 1 uses hard delete.
15. Activity/Audit may survive source deletion.
16. PostgreSQL owns MinIO business metadata.
17. External services cannot directly mutate business tables.
18. PROD PostgreSQL remains outside Kubernetes.
19. Replication does not replace backup.

# 21. Open Decisions
1. Identifier strategy.
2. SQL naming convention, data types, and timestamp/time-zone standard.
3. Physical Sprint-to-UserStory association.
4. Board placement/order representation.
5. Optimistic locking beyond UserStory.
6. Outbox claiming/cleanup and consumer deduplication persistence.
7. Foreign-key deletion rules.
8. MinIO reconciliation when object features exist.
9. Runtime versus migration DB identities.
10. Identity Reference lifecycle after Keycloak identity removal.