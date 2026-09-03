# ProjectBoard Network and Trust-Boundary Diagrams

[Back to Network Design](../network-design.md)

This document visualizes the main ProjectBoard network and trust boundaries.

```text
Default Deny + Explicitly Allowed Communication
```

# 1. Main Application Network
```mermaid
flowchart LR
    User[Browser]
    Gateway[Nginx Gateway]

    subgraph DEV[DEV]
        DevFE[Frontend]
        DevBE[Backend]
        DevPG[(PostgreSQL)]
        DevRedis[(Redis)]
        DevMQ[RabbitMQ]

        DevBE --> DevPG
        DevBE --> DevRedis
        DevBE --> DevMQ
    end

    subgraph STAGE[STAGE]
        StageFE[Frontend]
        StageBE[Backend]
        StagePG[(PostgreSQL)]
        StageRedis[(Redis)]
        StageMQ[RabbitMQ]

        StageBE --> StagePG
        StageBE --> StageRedis
        StageBE --> StageMQ
    end

    subgraph PROD[PROD Kubernetes]
        Ingress[Ingress]
        ProdFE[Frontend]
        ProdBE[Backend]
        ProdRedis[(Redis)]
        ProdMQ[RabbitMQ]

        Ingress --> ProdFE
        Ingress --> ProdBE
        ProdBE --> ProdRedis
        ProdBE --> ProdMQ
    end

    User -->|HTTPS| Gateway
    Gateway --> DevFE
    Gateway --> DevBE
    Gateway --> StageFE
    Gateway --> StageBE
    Gateway --> Ingress
```

Public traffic enters only through the Gateway.
DEV, STAGE, and PROD remain isolated.

# 2. Authentication and Secrets
```mermaid
flowchart LR
    Browser[Browser]
    Gateway[Nginx Gateway]

    DEV[DEV Backend]
    STAGE[STAGE Backend]
    PROD[PROD Backend]

    subgraph Security[Security VM]
        Keycloak[Keycloak]
        KeycloakDB[(Keycloak DB)]
        Vault[Vault]

        Keycloak --> KeycloakDB
    end

    Browser -->|HTTPS| Gateway
    Gateway -->|Public OIDC| Keycloak

    DEV -->|OIDC metadata / JWKS| Keycloak
    STAGE -->|OIDC metadata / JWKS| Keycloak
    PROD -->|OIDC metadata / JWKS| Keycloak

    DEV -.->|DEV secrets| Vault
    STAGE -.->|STAGE secrets| Vault
    PROD -.->|PROD secrets| Vault
```

Keycloak public authentication and administration use separate trust paths.
Lower environments cannot access higher-environment secrets.

# 3. Production Database Boundary
```mermaid
flowchart LR
    Backend[PROD Backend]
    Primary[(prod-db-primary)]
    Replica[(prod-db-replica)]
    Backup[Backup Destination]
    Prometheus[Prometheus]

    Backend --> Primary
    Primary -->|Replication| Replica
    Primary -.->|Backup| Backup
    Prometheus -.->|Metrics| Primary
    Prometheus -.->|Metrics| Replica
```

Production PostgreSQL remains outside Kubernetes.
The replica is not a backup.

# 4. Shared Storage
```mermaid
flowchart LR
    DEV[DEV Backend]
    STAGE[STAGE Backend]
    PROD[PROD Backend]
    MinIO[Storage VM / MinIO]

    DEV -.->|DEV scope| MinIO
    STAGE -.->|STAGE scope| MinIO
    PROD -.->|PROD scope| MinIO
```

Each environment uses separate MinIO credentials and scopes.

# 5. Observability Flows
Metrics:

```mermaid
flowchart LR
    Prometheus[Prometheus]
    Targets[Gateway / Backend / DB / Redis / RabbitMQ]

    Prometheus -->|scrape| Targets
```

Logs:

```mermaid
flowchart LR
    Sources[Hosts / Services]
    Alloy[Source-side Alloy]
    Loki[Loki]
    Grafana[Grafana]

    Sources --> Alloy
    Alloy -->|push| Loki
    Loki --> Grafana
```

Monitoring receives only required telemetry access.

# 6. Management Plane
```mermaid
flowchart LR
    Admin[Developer / Administrator]
    Entry[Controlled Management Entry]

    Entry --> SSH[SSH]
    Entry --> Keycloak[Keycloak Admin]
    Entry --> Vault[Vault Admin]
    Entry --> Grafana[Grafana]
    Entry --> MinIO[MinIO Admin]
    Entry --> PostgreSQL[PostgreSQL Admin]
    Entry --> K8s[Kubernetes API]

    Admin --> Entry
```

The exact management mechanism remains open.
Administrative interfaces are not public.

# 7. Future AI Boundary
```mermaid
flowchart LR
    Backend[ProjectBoard Backend]
    RabbitMQ[RabbitMQ]
    AI[AI Service]
    Ollama[Windows Host / Ollama]
    MinIO[MinIO]

    Backend --> RabbitMQ
    RabbitMQ --> AI
    AI --> Ollama
    AI -.->|Approved objects| MinIO
    AI -->|Result| RabbitMQ
```

AI is outside Version 1.

Prohibited:

```text
Browser ─X─► AI Service
Browser ─X─► Ollama
AI Service ─X─► ProjectBoard PostgreSQL
Ollama ─X─► ProjectBoard PostgreSQL
```

# 8. Prohibited Paths
| Source | Destination |
| --- | --- |
| Internet | PostgreSQL / Redis / RabbitMQ / Vault |
| Internet | SSH / Kubernetes API |
| Internet | MinIO / Monitoring administration |
| Internet | AI Service / Ollama |
| Browser | PostgreSQL / Redis / RabbitMQ / Vault |
| Browser | AI Service / Ollama |
| Gateway | PostgreSQL / Redis / RabbitMQ / Vault |
| DEV/STAGE | PROD data / secrets / messaging |
| AI Service | ProjectBoard PostgreSQL |
| Ollama | ProjectBoard PostgreSQL |

Anything not explicitly allowed is denied by default.

# 9. Fixed Rules
1. Public ProjectBoard traffic enters through the Gateway.
2. Public Keycloak OIDC traffic uses the Gateway.
3. DEV, STAGE, and PROD remain isolated.
4. PostgreSQL, Redis, RabbitMQ, and Vault are not public.
5. Kubernetes API and administrative interfaces are management-only.
6. Redis and RabbitMQ are environment-specific.
7. PROD PostgreSQL remains outside Kubernetes.
8. Replication does not replace backup.
9. Shared infrastructure uses environment-specific credentials and policies.
10. Monitoring receives only required telemetry connectivity.
11. AI and Ollama have no direct ProjectBoard database path.
12. Default deny applies to unlisted traffic.