# ProjectBoard Backup and Recovery Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Last updated | 2026-09-03 |
This document defines backup scope, restore expectations, and recovery verification. Exact commands, schedules, retention, and storage locations are implementation decisions.

# 1. Goals
Version 1 requires:
- scheduled backups;
- storage outside the protected source VM;
- basic integrity verification;
- documented manual restore;
- periodic isolated restore testing.
Automatic failover, PITR, WAL archiving, and advanced disaster recovery are future work.

# 2. Backup Scope
| Asset | Requirement |
| --- | --- |
| ProjectBoard PostgreSQL | Required |
| Keycloak DB | Required |
| MinIO data | Required when business objects are stored |
| Vault state | Required when Vault is operational |
| IaC/configuration | Recreated primarily from Git |
Redis is disposable. RabbitMQ is transport, not authoritative backup state. Unpublished event intent remains protected by the PostgreSQL Outbox.
The PROD PostgreSQL replica does not replace backup.

# 3. Backup Storage
Backups shall not exist only on the same VM as the protected service.
```text
Source Service → Backup Process → Separate Backup Destination
````

The destination may initially be host storage, the Storage VM, or another controlled location. Off-site copies are future work. Backup storage shall not be public.

# 4. Backup Workflow
A scheduled backup shall:

1. create the backup;
2. verify command success;
3. verify non-empty/readable output;
4. calculate or verify integrity information where practical;
5. store the backup outside the source service;
6. record success or failure.
   Useful metadata includes timestamp, environment, asset, backup identifier, size, and verification status.

# 5. Restore Verification
A created backup does not prove recoverability.
Periodic isolated restore tests should verify:

* database startup and expected schema;
* representative business data;
* Keycloak data where applicable;
* MinIO references where applicable;
* startup of a compatible application version.

```text
Backup Created ≠ Recovery Proven
```

# 6. Restore Flow
```text
Failure
  ↓
Select valid backup
  ↓
Recreate infrastructure if needed
  ↓
Restore persistent data
  ↓
Deploy compatible application version
  ↓
Validate and return service
```

Version 1 restore may be manual but shall be documented and reproducible.

# 7. PostgreSQL Recovery
ProjectBoard PostgreSQL is the business source of truth. Recovery restores a compatible database before normal Backend operation resumes.
Flyway compatibility must be considered when selecting the application version. A restored database shall not be blindly changed by incompatible migrations before verification.

# 8. Keycloak, MinIO, and Vault
Keycloak is a separate recovery unit. Recovery must restore the identity-provider state required by ProjectBoard; ProjectBoard shall not reconstruct it by directly modifying Keycloak tables.
When MinIO stores business objects, recovery must preserve usable PostgreSQL metadata-to-object references.
When Vault is operational, Vault state and required recovery material must be protected; recovery material shall not exist only inside the protected Vault instance.

# 9. Reproducible Infrastructure
Configuration is reconstructed primarily from:

```text
Git
├── Terraform
├── Ansible
├── Docker Compose
├── Kubernetes manifests
├── Flyway migrations
└── monitoring configuration
```

Secrets are excluded from Git and require their own recovery mechanism.

# 10. RabbitMQ and Redis Recovery
RabbitMQ may be recreated from configuration. Recoverable pending events in the PostgreSQL Outbox may be published after broker recovery. Consumers remain idempotent.
Redis may be recreated. Loss of Redis may remove locks, cache, or rate-limit state but shall not remove authoritative business data.

# 11. Backup Security
Backups contain sensitive data and require:

* restricted identities;
* protected transfer;
* non-public storage;
* controlled restore access;
* no secrets in backup logs;
* no unnecessary delete permission for application identities.
  Encryption at rest is applied where required by data sensitivity and storage location.

# 12. Observability
Backup operations shall expose:

* last successful backup;
* backup failure;
* backup age;
* restore-test status.
  Backup failure must not remain silent. Detailed dashboards and alerts belong to `observability-design.md`.

# 13. Recovery Objectives
Exact RPO and RTO remain open.

```text
RPO → acceptable recent data loss
RTO → acceptable recovery duration
```

Version 1 prioritizes reliable scheduled backup and tested manual recovery over strict SLA targets.

# 14. Fixed Rules
1. ProjectBoard PostgreSQL must be backed up.
2. Keycloak persistent state must be recoverable.
3. MinIO is backed up when it stores ProjectBoard business objects.
4. Vault requires recovery protection when operational.
5. Redis contains no authoritative business data.
6. RabbitMQ does not replace business backup.
7. Transactional Outbox protects unpublished event intent.
8. PostgreSQL replication does not replace backup.
9. Backups are stored outside the protected source VM.
10. Backup creation and failure are observable.
11. Restore procedures are documented.
12. Periodic restore verification is required.
13. Infrastructure is reproducible from version-controlled configuration.
14. Secrets are not committed to Git.
15. Automatic failover and PITR are outside Version 1.

# 15. Open Decisions
1. Backup schedule and retention.
2. Backup destination.
3. PostgreSQL and Keycloak backup mechanisms.
4. MinIO backup mechanism when active.
5. Vault backup/recovery mechanism.
6. Restore-test frequency.
7. Future RPO/RTO targets.
8. Whether off-site backup is introduced.
9. Whether PostgreSQL PITR/WAL archiving is introduced later.