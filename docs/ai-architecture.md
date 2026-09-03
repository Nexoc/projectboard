# Future AI Architecture
| Field | Value |
| --- | --- |
| Status | Future architecture |
| Release scope | Outside Version 1 |
| Parent | [High-Level Design](hld.md) |
| Last updated | 2026-09-03 |
This document defines the future AI boundary. AI remains independent from the ProjectBoard core and must never be required for normal application operation.

# 1. Goals
Future AI may support:
- User Story analysis;
- screenshot/image analysis;
- suggested Tasks, descriptions, and acceptance criteria;
- asynchronous analysis jobs.
Model-specific logic remains outside Spring Boot business modules.

# 2. Non-Goals
AI shall not:
- directly modify ProjectBoard business data;
- be called directly by the browser;
- expose Ollama publicly;
- access ProjectBoard PostgreSQL directly;
- create real Tasks without explicit user confirmation;
- become a dependency of core ProjectBoard availability;
- treat model output as trusted data.

# 3. Components
| Component | Responsibility |
| --- | --- |
| Frontend | Starts analysis and presents suggestions |
| Backend | Authorization, job ownership, validation, acceptance |
| PostgreSQL | Authoritative AI job/result metadata |
| RabbitMQ | Async request/result transport |
| Transactional Outbox | Reliable request publication |
| AI Service | Orchestration and structured result generation |
| MinIO | Binary AI input when required |
| Ollama / Vision | Local inference |
| Redis | Optional temporary coordination |
The AI Service owns no Projects, memberships, UserStories, Tasks, Sprints, or ProjectBoard authorization.

# 4. Boundary
```text
Frontend → Backend → RabbitMQ → AI Service → Ollama / Vision
````

Prohibited:

```text
Frontend ─X─► AI Service
Frontend ─X─► Ollama
Internet ─X─► Ollama
AI Service ─X─► ProjectBoard PostgreSQL
```

The Backend remains the authoritative business boundary.

# 5. Request Flow
```text
User
 ↓
Backend API
 ↓
Authenticate + Authorize + Validate
 ↓
Create AI Job + Outbox Event
 ↓
PostgreSQL COMMIT
 ↓
Outbox Publisher
 ↓
RabbitMQ
 ↓
AI Service
 ↓
Ollama / Vision
```

The AI job and request event are persisted atomically. RabbitMQ failure leaves the request recoverable from the Outbox.

# 6. Result Flow
```text
AI Service
 ↓
AIAnalysisCompleted
 ↓
RabbitMQ
 ↓
ProjectBoard Consumer
 ↓
PostgreSQL
 ↓
Frontend
```

AI results are suggestions, not business mutations.

# 7. User Confirmation
```text
AI Suggestion
   ↓
User Review
 ├── Reject
 └── Accept
       ↓
ProjectBoard Backend
       ↓
Normal business use case
```

Accepted suggestions still require current authorization, validation, and concurrency checks. Real Tasks are created only through the Backend.

# 8. AI Job State
A possible lifecycle is:

```text
REQUESTED → PROCESSING → SUGGESTED → ACCEPTED
                    └──→ FAILED
                    └──→ REJECTED
```

An optional `EXPIRED` state may be introduced later.
PostgreSQL remains authoritative for ProjectBoard-visible job state. Redis may hold temporary execution state only.

# 9. Messaging Reliability
AI messaging follows existing ProjectBoard rules:

* Transactional Outbox;
* at-least-once delivery;
* idempotent consumers;
* bounded retry;
* DLQ;
* unique `eventId`;
* propagated `correlationId`.

Duplicate delivery must not create duplicate jobs, suggestions, or Tasks.
Detailed semantics belong to `messaging-design.md`.

# 10. Binary Input
When screenshots or images are required:

```text
PostgreSQL metadata → MinIO object → AI Service
```

PostgreSQL owns Project/object authorization metadata; MinIO stores bytes.
The AI Service receives only object access required for the specific job.
Exact upload and object-access mechanisms remain open.

# 11. Environment Isolation
DEV, STAGE, and PROD keep separate AI job data, RabbitMQ resources, MinIO scopes, credentials, and policies.
Cross-environment AI access is prohibited.

# 12. AI Service Responsibilities
The AI Service may:

* consume approved AI requests;
* prepare model input;
* retrieve approved objects;
* invoke Ollama/Vision;
* validate and structure model output;
* publish results or failures.

It shall not perform Project authorization or directly create/modify ProjectBoard entities.

# 13. Model Runtime
Ollama/Vision runs on the GPU-equipped Windows host.

```text
AI Service → controlled internal route → Windows Host → Ollama / Vision → GPU
```

Ollama is not public. Exact authentication, bind address, TLS, and Windows firewall rules remain implementation decisions.

# 14. Output Contract
AI returns structured versioned data rather than executable commands.

A result may identify:

* `jobId`, `eventId`, `correlationId`;
* aggregate/project reference;
* schema version;
* model identifier;
* suggestion fields;
* processing status/errors;
* creation timestamp.

Model output must be parsed and validated against an allowed structure before use.

# 15. Security and Privacy
AI output and uploaded content are untrusted input.
Controls shall address prompt injection, malformed output, file/content validation, Project isolation, least-privilege MinIO access, data minimization, and retention.

Prompts shall not contain passwords, tokens, Vault credentials, database credentials, or unrelated Project data.
Technical logs shall not store complete prompts or binary input by default.

# 16. Failure and Staleness
AI failures degrade only AI functionality.

| Failure                       | Result                                |
| ----------------------------- | ------------------------------------- |
| AI Service/Ollama unavailable | Job fails/retries; core continues     |
| RabbitMQ unavailable          | Outbox retains request                |
| MinIO unavailable             | Object-dependent jobs fail/wait       |
| Invalid model output          | Result rejected                       |
| Duplicate message             | Idempotency prevents duplicate effect |
| Source Story changed          | Acceptance revalidates current state  |

Long-running analysis may become stale. Acceptance shall revalidate membership, authorization, current entity state, and relevant version.

# 17. Observability
Future telemetry may include job counts, failures, queue wait, processing/inference duration, retries, DLQ activity, model availability, and object-access failures.
Telemetry includes environment and correlation context but excludes prompts/user content by default.
Model-quality evaluation is separate from infrastructure monitoring.

# 18. Versioning
Where practical, results should identify:

```text
modelVersion
promptTemplateVersion
resultSchemaVersion
```

This supports reproducibility and controlled upgrades. Automatic retraining/model updates are outside this architecture.

# 19. Fixed Rules
1. AI is outside Version 1.
2. AI is a separate service.
3. Frontend reaches AI only through ProjectBoard Backend.
4. AI Service never directly modifies ProjectBoard business data.
5. Ollama is never publicly exposed.
6. Backend authorizes every AI request.
7. RabbitMQ is used for asynchronous AI processing.
8. AI requests use the Transactional Outbox.
9. AI consumers are idempotent and use retry/DLQ.
10. AI output is untrusted structured suggestion data.
11. Real Tasks require explicit user confirmation.
12. Real business changes use normal Backend use cases.
13. PostgreSQL owns authoritative AI job metadata.
14. Redis is temporary only.
15. MinIO may hold binary AI input.
16. AI failure does not disable core ProjectBoard.
17. AI data and infrastructure remain environment-isolated.
18. Suggestions are revalidated before acceptance.

# 20. Open Decisions
1. Release that introduces AI.
2. Initial model(s).
3. AI result schema and persisted job fields.
4. Browser-to-MinIO upload flow.
5. File scanning/validation controls.
6. AI Service authentication to MinIO and Ollama.
7. Internal inference TLS/authentication.
8. AI job/object/suggestion retention.
9. Partial acceptance representation.
10. Stale-suggestion policy.