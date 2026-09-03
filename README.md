# ProjectBoard DevSecOps Lab

A small Kanban project-management application built primarily as a hands-on DevOps and DevSecOps learning project.

> The project code is written by me.
> AI is used as a technical mentor to explain concepts, review decisions, and help me understand problems — not to automatically write the project for me.

## Goal

The main principle is:

```text
Simple Business Logic
+
Progressively Complex Infrastructure
````

The application itself stays relatively simple so the project can focus on learning architecture, deployment, automation, security, observability, and infrastructure.

## Application Stack

* Backend: Spring Boot
* Frontend: Angular
* Database: PostgreSQL
* Database migrations: Flyway
* Authentication: Keycloak
* Cache / temporary state: Redis
* Messaging: RabbitMQ
* Object storage: MinIO
* API: REST + OpenAPI

## DevOps / DevSecOps Topics

The project is used to learn and practice:

* Git and GitHub
* Linux
* Docker
* Docker Compose
* Networking
* Nginx / Reverse Proxy
* OAuth 2.0 / OpenID Connect / JWT
* TLS
* Secrets management
* CI/CD
* Infrastructure as Code
* Terraform
* Ansible
* Monitoring and Logging
* Security and DevSecOps
* Backup and Recovery
* Kubernetes
* GitOps / Argo CD

## Environments

```text
DEV   → Docker Compose
STAGE → Docker Compose
PROD  → Kubernetes target
```

DEV, STAGE, and PROD use isolated data, configuration, credentials, and supporting infrastructure.

## Development Approach

The project is implemented incrementally.

Each major phase should be:

1. understood;
2. implemented;
3. tested;
4. documented;
5. reproducible.

See [`docs/roadmap.md`](docs/roadmap.md) for the implementation order.

## Architecture

The Backend is a Spring Boot Modular Monolith.

Architecture and requirements are documented in:

* [`docs/srs.md`](docs/srs.md)
* [`docs/hld.md`](docs/hld.md)
* [`docs/`](docs/)

The API contract is maintained in:

* [`api/openapi.yaml`](api/openapi.yaml)

## Future

Outside Version 1:

* AI / local LLM integration
* Ollama / Vision models
* GitOps automation
* advanced high availability
* native Android / iOS applications