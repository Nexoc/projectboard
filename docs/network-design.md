# ProjectBoard Network Design
| Field | Value |
| --- | --- |
| Status | Working architecture baseline |
| Parent | [High-Level Design](hld.md) |
| Security | [Security Architecture](security-architecture.md) |
| Diagram | [Network Diagram](diagrams/network-diagram.md) |
| Last updated | 2026-09-03 |

This document defines network zones, allowed communication, isolation, ingress, and management boundaries.
Exact IPs, CIDRs, firewall syntax, interface names, and Kubernetes CNI details belong to infrastructure implementation.

# 1. Principles
1. Default deny.
2. Allow only communication required by an explicit responsibility.
3. Public application traffic enters through the Gateway.
4. DEV, STAGE, and PROD remain isolated.
5. Internal infrastructure is not publicly exposed.
6. Network reachability never replaces authentication or Backend authorization.
7. Administrative traffic is separated from user traffic.
8. Lower environments cannot access PROD data, secrets, or messaging state.
9. Monitoring receives only required telemetry connectivity.
10. Future AI has no direct ProjectBoard database path.

# 2. Network Zones
| Zone | Components | Purpose |
| --- | --- | --- |
| External | Internet, browsers | Untrusted traffic |
| Public Entry | Nginx Gateway | HTTPS ingress |
| DEV | DEV Compose services | Development |
| STAGE | STAGE Compose services | Validation |
| PROD | Kubernetes workloads | Production target |
| PROD DB | PostgreSQL primary/replica | Persistence |
| Security | Keycloak, Keycloak DB, Vault | Identity/secrets |
| Storage | MinIO | Object storage |
| Monitoring | Prometheus, Grafana, Loki, Alloy, Alertmanager | Observability |
| Management | Admin path | Infrastructure administration |
| AI | Future AI Service, Ollama | Future inference |

# 3. Environment Isolation
```text
dev.project.davl.at   → DEV
stage.project.davl.at → STAGE
project.davl.at       → PROD
````

Each environment has separate data, credentials, configuration, Redis state, RabbitMQ deployment/state, MinIO scope, and identity-provider configuration.
DEV and STAGE shall not access PROD data, secrets, Redis, RabbitMQ, or storage scopes.

# 4. Public Entry
```text
Internet
   │ HTTPS
   ▼
Gateway
   ├── DEV
   ├── STAGE
   ├── PROD
   └── Keycloak public OIDC route
```

The `gateway` VM runs Nginx for TLS termination, reverse proxying, hostname routing, and basic rate limiting.
TCP 443 is the normal public listener. TCP 80 is limited to redirect or certificate automation.

The following are never publicly exposed: PostgreSQL, Redis, RabbitMQ, Vault, MinIO administration, Monitoring administration, Kubernetes API, SSH, AI Service, and Ollama.
Keycloak administration remains on the management path.

# 5. Application Paths
Normal paths are:

```text
Browser → Gateway → Frontend / Backend
Browser → Gateway → Keycloak OIDC
Backend → PostgreSQL / Redis / RabbitMQ
Backend → Keycloak metadata/keys
Approved workload → Vault
Backend → MinIO
```

Every dependency is environment-specific.
The Browser never receives direct infrastructure access or credentials.

# 6. Database, Redis, and RabbitMQ
DEV and STAGE PostgreSQL remain in their Compose environments.
PROD uses:

```text
PROD Backend → prod-db-primary → prod-db-replica
```

Only approved application, replication, backup, monitoring, and administrative paths may reach PROD PostgreSQL.

Redis:

```text
DEV Backend   → DEV Redis
STAGE Backend → STAGE Redis
PROD Backend  → PROD Redis
```

RabbitMQ:

```text
DEV   → DEV Compose RabbitMQ
STAGE → STAGE Compose RabbitMQ
PROD  → Kubernetes RabbitMQ
```

RabbitMQ management is restricted. Messaging semantics belong to `messaging-design.md`.

# 7. Shared Security and Storage
Approved flows include environment workloads to required Keycloak endpoints, matching Vault scopes, and matching MinIO scopes.
DEV/STAGE shall not access PROD Vault or MinIO scopes.
Vault and MinIO administration use the management path.
The Frontend never receives unrestricted MinIO credentials.

# 8. Observability Network
Metrics:

```text
Prometheus → scrape targets
```

Logs:

```text
Source → source-side Alloy → Loki
```

Monitoring connectivity does not grant shell, database-admin, Vault-admin, or unrestricted lateral access.
Detailed telemetry belongs to `observability-design.md`.

# 9. Management Plane
```text
Developer / Administrator
          ↓
Controlled Management Path
          ├── SSH / Gateway
          ├── Keycloak / Vault
          ├── PostgreSQL / MinIO
          ├── Monitoring
          └── Kubernetes API
```

The exact mechanism remains open and may use VPN, private management networking, bastion access, or another controlled solution.
SSH is administrative, not a public application endpoint.

# 10. Allowed Flows
| Source             | Destination                 | Purpose            |
| ------------------ | --------------------------- | ------------------ |
| Internet           | Gateway :443                | Public HTTPS       |
| Internet           | Gateway :80                 | Redirect/ACME only |
| Gateway            | DEV/STAGE/PROD app          | Reverse proxy      |
| Gateway            | Keycloak public OIDC        | Authentication     |
| Backend            | Same-environment PostgreSQL | Persistence        |
| Backend            | Same-environment Redis      | Temporary state    |
| Publisher/Consumer | Same-environment RabbitMQ   | Messaging          |
| Backend            | Keycloak metadata/keys      | Token support      |
| Approved workload  | Matching Vault scope        | Secrets            |
| Backend            | Matching MinIO scope        | Objects            |
| Prometheus         | Metrics targets             | Metrics            |
| Source Alloy       | Loki                        | Logs               |
| Primary DB         | Replica DB                  | Replication        |
| Management path    | Admin endpoints             | Administration     |
| CI/CD identity     | Deployment endpoint         | Deployment         |

Anything not explicitly allowed is denied by default.

# 11. Prohibited Paths
| Source     | Destination                           |
| ---------- | ------------------------------------- |
| Internet   | PostgreSQL / Redis / RabbitMQ / Vault |
| Internet   | SSH / Kubernetes API                  |
| Internet   | MinIO / Monitoring administration     |
| Internet   | AI Service / Ollama                   |
| Browser    | PostgreSQL / Redis / RabbitMQ / Vault |
| Browser    | AI Service / Ollama                   |
| Gateway    | PostgreSQL / Redis / RabbitMQ / Vault |
| DEV/STAGE  | PROD data / secrets / messaging       |
| AI Service | ProjectBoard PostgreSQL               |
| Ollama     | ProjectBoard PostgreSQL               |

# 12. DEV/STAGE Compose
DEV and STAGE use private Compose networks where practical.
PostgreSQL, Redis, and RabbitMQ do not require public host-port exposure.
Only components requiring approved external communication bind host interfaces.

# 13. PROD Kubernetes
```text
Internet → Gateway → Kubernetes Ingress → Frontend / Backend
```

PROD PostgreSQL remains outside Kubernetes and the Kubernetes API is management-only.
The Kubernetes phase uses NetworkPolicies with a default-deny baseline and explicit required flows.
Exact CNI, namespaces, CIDRs, ingress, and NetworkPolicy implementation remain open.

# 14. TLS and Egress
Public traffic always uses HTTPS.
Universal internal mTLS is not required initially; internal TLS is introduced where justified by trust boundaries or sensitive credential/token transport.
Runtime Internet egress should not be unrestricted. Required DNS, NTP, registry, package, certificate, alert, and backup traffic should be documented and restricted where practical.

# 15. CI/CD and Backup
CI/CD uses a restricted deployment identity and approved endpoint; the exact direct-deployment path remains open.
Future GitOps reduces broad direct PROD access.

Backup traffic uses:

```text
Protected Source → Backup Process → Backup Destination
```

Backup storage is never public. Detailed transfer design belongs to `backup-recovery.md`.

# 16. Future AI
AI is outside Version 1.

```text
ProjectBoard → RabbitMQ → AI Service → controlled route → Ollama
```

AI Service and Ollama are not public.
The Frontend does not call AI directly.
AI receives no ProjectBoard database credentials and cannot directly mutate business state.

# 17. Verification
Verify both allowed and denied communication:

* public HTTPS works;
* required internal dependencies work;
* DEV/STAGE → PROD access fails;
* internal services are not public;
* Browser cannot reach internal data services;
* Keycloak login works while Keycloak Admin remains restricted;
* Kubernetes API is management-only;
* Docker published ports match the design.

# 18. Fixed Rules
1. Default deny is the baseline.
2. Public application traffic enters through the Gateway using HTTPS.
3. DEV, STAGE, and PROD remain isolated.
4. Lower environments cannot access PROD data, secrets, or messaging.
5. PostgreSQL, Redis, RabbitMQ, and Vault are never public.
6. MinIO and Monitoring administration are never public.
7. Kubernetes API and SSH are management-only.
8. Public Keycloak OIDC uses the Gateway; administration uses the management path.
9. Gateway has no direct PostgreSQL, Redis, RabbitMQ, or Vault access.
10. Monitoring receives only required telemetry connectivity.
11. Browser code receives no infrastructure credentials.
12. PROD PostgreSQL remains outside Kubernetes.
13. Kubernetes uses explicit workload network restrictions.
14. Future AI has no direct ProjectBoard database access.
15. Ollama is never publicly exposed.
16. Network configuration should become reproducible through IaC.

# 19. Open Decisions
1. VirtualBox network layout, VM IPs, and CIDRs.
2. Management-access mechanism.
3. Gateway-to-application internal protocol.
4. Internal TLS boundaries and PKI.
5. Telemetry authentication and transport.
6. Runtime operational egress rules.
7. CI/CD deployment network path.
8. Kubernetes CNI, ingress, namespaces, and NetworkPolicies.
9. Windows/Ollama internal route and protection.
10. Backup transfer protocol and destination path.