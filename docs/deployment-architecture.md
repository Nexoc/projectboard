# Deployment Architecture
| Field | Value |
| --- | --- |
| Status | Draft |
| Scope | DEV/STAGE baseline and PROD Kubernetes target |
| Parent | [High-Level Design](hld.md) |
| Diagram | [Deployment Diagram](diagrams/deployment-diagram.md) |
| Last updated | 2026-09-03 |
This document defines where ProjectBoard components run and how deployment evolves from Docker Compose to Kubernetes. Operational commands belong to implementation runbooks.

# 1. Principles
1. DEV and STAGE use Docker Compose; PROD targets Kubernetes.
2. Application images are built once and promoted unchanged.
3. Configuration and secrets remain outside images.
4. Durable state remains outside disposable application containers.
5. Backend remains stateless where practical.
6. DEV, STAGE, and PROD data and credentials are isolated.
7. Terraform provisions infrastructure; Ansible configures hosts.
8. Backup and restore precede advanced HA.

# 2. Environment Model
| Environment | Purpose | Deployment |
| --- | --- | --- |
| DEV | Development/integration | Dedicated VM + Docker Compose |
| STAGE | Production-like validation | Dedicated VM + Docker Compose |
| PROD | Production target | Three-node Kubernetes |
Each environment has separate business data, configuration, credentials, secrets, Redis state, RabbitMQ deployment/state, and storage scopes.

# 3. Deployable Components
| Component | Placement |
| --- | --- |
| Angular Frontend | DEV/STAGE Compose; PROD Kubernetes |
| Spring Boot Backend | DEV/STAGE Compose; PROD Kubernetes |
| PostgreSQL | DEV/STAGE Compose; dedicated PROD VMs |
| Redis | Environment-specific |
| RabbitMQ | Environment-specific |
| Nginx | `gateway` VM |
| Keycloak + Keycloak DB + Vault | `security` VM |
| MinIO | `storage` VM |
| Monitoring stack | `monitoring` VM |
| Ollama / Vision | Future Windows GPU host |
Outbox Publisher and Activity/Audit Consumer run inside the Spring Boot Backend in Version 1.

# 4. DEV
```text
dev VM
└── Docker Compose
    ├── Angular Frontend
    ├── Spring Boot Backend
    ├── PostgreSQL
    ├── Redis
    └── RabbitMQ
````

DEV owns its own data, configuration, credentials, Redis state, and RabbitMQ state. Shared infrastructure is accessed through environment-specific identities and scopes.

# 5. STAGE

```text
stage VM
└── Docker Compose
    ├── Angular Frontend
    ├── Spring Boot Backend
    ├── PostgreSQL
    ├── Redis
    └── RabbitMQ
```

STAGE is isolated from DEV and PROD and receives the same immutable artifact validated in DEV.

# 6. PROD Target

The accepted PROD target is:

```text
k8s-node-1
k8s-node-2
k8s-node-3
```

Kubernetes runs Frontend, Backend, Redis, RabbitMQ, and required supporting workloads. Production PostgreSQL remains outside Kubernetes. Exact Kubernetes distribution, CNI, ingress, namespaces, and NetworkPolicies remain open.

# 7. Production PostgreSQL

```text
PROD Backend
     ↓
prod-db-primary
     ↓ replication
prod-db-replica
```

The primary is the normal application endpoint. The replica supports replication and recovery learning. Automatic failover is future work. Replication does not replace backup.

# 8. Shared Infrastructure

```text
gateway
└── Nginx

security
├── Keycloak
├── Keycloak DB
└── Vault

storage
└── MinIO

monitoring
├── Prometheus
├── Grafana
├── Loki
├── Grafana Alloy
└── Alertmanager
```

Shared Security, Storage, and Monitoring infrastructure is accepted for the home lab. Environment separation remains logical and credential-based.

# 9. VM Topology

```text
Windows Host
├── gateway
├── security
├── storage
├── dev
├── stage
├── k8s-node-1
├── k8s-node-2
├── k8s-node-3
├── prod-db-primary
├── prod-db-replica
└── monitoring
```

All Linux VMs use Debian 13. The Windows host also provides the future GPU runtime.

# 10. Initial Resources

| VM                                                             | CPU |  RAM |  Disk |
| -------------------------------------------------------------- | --: | ---: | ----: |
| gateway                                                        |   1 | 2 GB | 20 GB |
| security                                                       |   2 | 4 GB | 30 GB |
| storage                                                        |   1 | 2 GB | 40 GB |
| dev                                                            |   2 | 4 GB | 40 GB |
| stage                                                          |   2 | 4 GB | 40 GB |
| k8s-node-1                                                     |   2 | 4 GB | 40 GB |
| k8s-node-2                                                     |   2 | 4 GB | 40 GB |
| k8s-node-3                                                     |   2 | 4 GB | 40 GB |
| prod-db-primary                                                |   1 | 4 GB | 40 GB |
| prod-db-replica                                                |   1 | 4 GB | 40 GB |
| monitoring                                                     |   2 | 6 GB | 50 GB |
| Resources are initial values and may change after measurement. |     |      |       |

# 11. Physical Host

Current host: Windows, Ryzen 7 3700X (8C/16T), 96 GB RAM, NVIDIA RTX 3060 16 GB, 2 TB NVMe, VirtualBox. Controlled CPU overcommit is acceptable for the home lab.

# 12. Persistence

Application containers are disposable. Persistent state includes ProjectBoard PostgreSQL, Keycloak DB, MinIO when active, and Vault when operational. Redis is temporary. RabbitMQ is transport, not authoritative business storage. Recovery details belong to `backup-recovery.md`.

# 13. Configuration and Secrets

```text
Same Image
├── DEV config + secrets
├── STAGE config + secrets
└── PROD config + secrets
```

Environment-specific application code is prohibited. Secrets remain outside Git and images. Vault is the centralized target secrets platform.

# 14. Infrastructure as Code

```text
Terraform → VM Infrastructure → Ansible → OS/Server Roles → Compose/Kubernetes
```

Terraform owns infrastructure lifecycle. Ansible owns guest OS and server-role configuration. Terraform outputs should support Ansible inventory. The architecture does not depend on one VirtualBox provider.

# 15. Artifact Promotion

```text
Git Commit → GitHub Actions → Build/Test/Scan → GHCR → DEV → STAGE → Approval → PROD
```

Frontend and Backend images are built once and promoted unchanged. Environment configuration and secrets are injected separately. Terraform and Ansible do not run on every application deployment. Detailed delivery belongs to `cicd-design.md`.

# 16. Health and Failure

Deployment exposes liveness, readiness, and dependency health. PostgreSQL failure affects durable operations; Redis failure degrades temporary features; RabbitMQ failure delays async work while Outbox retains intent; monitoring failure does not stop business processing; future AI failure does not affect core availability. Details belong to `observability-design.md`.

# 17. GitOps and HA

```text
Manual Deployment → CI/CD → Direct Kubernetes → GitOps → Argo CD
```

Argo CD runs inside Kubernetes. GitOps follows direct Kubernetes deployment. Gateway HA and automatic PostgreSQL failover are future phases.

# 18. Future AI

AI is outside Version 1.

```text
ProjectBoard → RabbitMQ → AI Service → Windows Host / Ollama / Vision / GPU
```

The AI Service is separate from Spring Boot. Ollama remains internal. Detailed placement belongs to `ai-architecture.md`.

# 19. Fixed Rules

1. DEV uses a dedicated Docker Compose VM.
2. STAGE uses a dedicated Docker Compose VM.
3. PROD targets a three-node Kubernetes cluster.
4. PROD PostgreSQL remains outside Kubernetes.
5. PROD uses primary and replica PostgreSQL VMs.
6. Replication does not replace backup.
7. Redis and RabbitMQ are environment-specific.
8. Outbox Publisher and Activity/Audit Consumer run in-process in Version 1.
9. Keycloak and Vault share the Security VM.
10. MinIO runs on the Storage VM.
11. Central observability runs on the Monitoring VM.
12. Nginx Gateway is the public entry point.
13. Linux VMs use Debian 13.
14. Terraform provisions infrastructure; Ansible configures hosts.
15. Application images are environment-independent and promoted unchanged.
16. Persistent state survives application-container replacement.
17. Backend remains stateless where practical.
18. GitOps follows direct Kubernetes deployment.
19. AI and automatic HA remain outside Version 1.

# 20. Open Decisions

1. VirtualBox Terraform provider.
2. Exact VirtualBox network/IP layout.
3. Management-access mechanism.
4. Internal TLS boundaries.
5. Kubernetes distribution, CNI, ingress, namespaces, and NetworkPolicies.
6. Vault bootstrap/deployment mechanism.
7. MinIO deployment details.
8. Resource adjustments after measurements.
9. Project phase when PROD Kubernetes becomes active.
10. Future Gateway HA and PostgreSQL failover technologies.