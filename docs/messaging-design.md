# ProjectBoard Messaging Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Persistence | [Database Design](database-design.md) |
| Last updated | 2026-09-03 |

This document defines Version 1 asynchronous messaging behavior.
RabbitMQ and the Transactional Outbox Pattern are mandatory.
Core business operations remain synchronous REST operations.

# 1. Goals
Messaging shall provide reliable publication of committed business events, asynchronous Activity/Audit processing, at-least-once delivery, idempotent consumption, bounded retry, DLQ handling, correlation, and DEV/STAGE/PROD isolation.
RabbitMQ is transport, not a business source of truth.

# 2. Version 1 Flow
```text
Client
  ↓
Spring Boot Backend
  ↓
PostgreSQL Transaction
├── Business State
└── Outbox Event
        ↓
      COMMIT
        ↓
Outbox Publisher
        ↓
RabbitMQ
        ↓
Activity/Audit Consumer
        ↓
PostgreSQL Activity/Audit
````

PostgreSQL provides durability for business state and publication intent.
RabbitMQ never participates in the PostgreSQL transaction.

# 3. Transactional Outbox

A business operation producing an asynchronous event shall persist the business mutation and Outbox event in the same PostgreSQL transaction.

```text
BEGIN
Business Change
Insert Outbox Event
COMMIT
```

If the transaction rolls back, neither change exists.
If RabbitMQ is unavailable after commit, the Outbox event remains pending.
Persistence details belong to `database-design.md`.

# 4. Event Contract

Each event shall contain at least:

* `eventId`, `eventType`, `schemaVersion`, `occurredAt`;
* `aggregateType`, `aggregateId`, `projectId`;
* `correlationId` where available;
* `payload`.

Example event types:

```text
ProjectCreated
MemberAdded
UserStoryCreated
TaskCompleted
```

Rules:

* `eventId` is unique;
* event names describe completed facts;
* contracts are versioned;
* additive compatible evolution is preferred;
* secrets and authentication tokens are prohibited from payloads.

# 5. Outbox Publisher

The Outbox Publisher runs inside the Spring Boot Backend process in Version 1.
It shall:

1. find and claim a bounded batch of pending records;
2. publish them to RabbitMQ;
3. wait for broker publication confirmation;
4. mark confirmed records published;
5. leave failed records retryable.

An event must not be marked published before broker confirmation.
A crash after broker acceptance but before the database update may cause duplicate publication; this is expected under at-least-once delivery.

# 6. Delivery and Idempotency

ProjectBoard uses at-least-once delivery.
Exactly-once delivery is not guaranteed.
Strict global event ordering is not required.
Duplicates may result from retries, crashes, acknowledgement failures, restarts, or replay.

The Activity/Audit Consumer also runs inside the Spring Boot Backend process.
A duplicate `eventId` must not create a duplicate logical effect.
Persistent deduplication may be stored transactionally with the consumer result.
The physical deduplication strategy belongs to `database-design.md`.

# 7. Activity/Audit

The initial consumer materializes business Activity/Audit records.
Business Activity/Audit is:

* asynchronous and eventually consistent;
* append-only through normal business APIs;
* separate from technical logs;
* separate from security logs.

RabbitMQ or consumer failure may delay audit creation but shall not roll back committed business state.
Hard-delete events shall contain enough historical information for processing after the source row no longer exists.

# 8. Retry and DLQ

Temporary consumer failures shall use bounded retry.
Repeated failures shall move the message to a Dead Letter Queue.

```text
Main Queue
   ↓
Consumer
   ↓ failure
Retry
   ↓ limit
DLQ
```

DLQ replay shall be controlled: inspect, fix the cause, replay selected messages, and verify the result.
Blind replay of all DLQ messages is not required.
Original `eventId` and `correlationId` shall remain available during replay.
Exact retry count, delay, backoff, and DLQ retention remain open.

# 9. RabbitMQ Topology

Initial logical topology:

```text
Domain Event Exchange
        ↓
Activity/Audit Queue
        ↓
Activity/Audit Consumer
```

Future independent consumers receive separate queues.
Exact exchange names, queue names, routing keys, queue types, and limits remain implementation decisions.

# 10. Environment Isolation

RabbitMQ is deployed separately by environment:

```text
DEV   → RabbitMQ in DEV Compose
STAGE → RabbitMQ in STAGE Compose
PROD  → RabbitMQ in Kubernetes
```

Each environment has independent credentials, exchanges, queues, messages, and DLQs.
DEV and STAGE shall never publish to or consume PROD messaging resources.

# 11. Failure Semantics

| Failure                       | Required Result                           |
| ----------------------------- | ----------------------------------------- |
| Business transaction rollback | No business mutation and no Outbox event  |
| RabbitMQ unavailable          | Business may commit; Outbox stays pending |
| Publisher failure             | Event remains retryable                   |
| Duplicate publication         | Consumer handles it safely                |
| Temporary consumer failure    | Bounded retry                             |
| Repeated consumer failure     | Message reaches DLQ                       |
| Consumer unavailable          | Business continues; audit is delayed      |
| Monitoring unavailable        | Messaging continues                       |

Complete RabbitMQ data loss does not corrupt PostgreSQL business state.
Only events recoverable from retained Outbox/source state can be republished.
Recovery of every previously published broker message is not guaranteed.

# 12. Correlation
The same `correlationId` should propagate through:

```text
HTTP Request
→ Backend
→ Outbox
→ RabbitMQ
→ Consumer
→ Activity/Audit
```

Structured logs should include `eventId`, `eventType`, `correlationId`, queue, and status where useful.
`eventId` and `correlationId` shall not be high-cardinality Prometheus labels.

# 13. Security
RabbitMQ is internal and shall not be publicly exposed.
Messaging access uses environment-specific credentials, least privilege, restricted network paths, and approved secret injection.
RabbitMQ administration is restricted to technical administrators.
Event and DLQ payloads retain the sensitivity of their business data.
Detailed controls belong to `security-architecture.md` and `network-design.md`.

# 14. Observability
Messaging observability shall detect:

* RabbitMQ unavailability and queue growth;
* missing consumers and consumer failures;
* Outbox publication failures, backlog, and oldest pending age;
* retries and DLQ messages.

Dashboards, retention, thresholds, and alert routing belong to `observability-design.md`.

# 15. Deployment
```text
Spring Boot Backend
├── REST API
├── Outbox Publisher
└── Activity/Audit Consumer

RabbitMQ
└── separate infrastructure service
```

Publisher and Activity/Audit Consumer are not separate deployable services in Version 1.
Extraction is allowed later only for a concrete scaling, deployment, or fault-isolation requirement.

# 16. Future AI
AI is outside Version 1.
A future flow may use `Backend → Outbox → RabbitMQ → AI Service → Ollama/Vision`.
AI results must return through controlled messaging and normal Backend use cases.
AI shall not directly modify ProjectBoard business data.

# 17. Fixed Rules
1. RabbitMQ is mandatory in Version 1.
2. Transactional Outbox is mandatory for asynchronous business events.
3. Business mutation and Outbox creation are atomic.
4. PostgreSQL remains the business source of truth.
5. RabbitMQ is transport only.
6. Delivery is at least once and consumers are idempotent.
7. Exactly-once delivery is not promised.
8. Strict global ordering is not required.
9. Events have unique `eventId` values.
10. Relevant flows propagate `correlationId`.
11. Broker confirmation precedes marking publication successful.
12. Retry is bounded and repeated failures reach a DLQ.
13. DLQ replay is controlled.
14. Activity/Audit is asynchronously materialized.
15. Consumer failure does not roll back committed business state.
16. Hard-delete events contain sufficient historical data.
17. DEV, STAGE, and PROD RabbitMQ deployments are isolated.
18. RabbitMQ is not publicly exposed.
19. Publisher and Activity/Audit Consumer run in-process in Version 1.

# 18. Open Decisions
1. Exchange, queue, and routing-key naming.
2. Classic versus quorum queues.
3. Retry count and backoff.
4. DLQ retention and replay tooling.
5. Outbox batch size, claiming, and published-record retention.
6. Event serialization and schema-versioning convention.
7. Consumer-deduplication implementation.