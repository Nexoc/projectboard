# CI/CD Design
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Deployment | [Deployment Architecture](deployment-architecture.md) |
| Last updated | 2026-09-03 |
This document defines CI validation, artifact creation, promotion, approval, verification, and rollback.

# 1. Delivery Model
```text
Branch → Pull Request → CI → Build → GHCR → DEV → STAGE → PROD
````

Frontend and Backend images are built once and promoted unchanged.

# 2. Git Workflow
```text
main       → stable/completed
dev        → integration
feature/*  → feature work
docs/*     → documentation
infra/*    → infrastructure
security/* → security work
fix/*      → fixes
```

Normal flow:

```text
Branch → Pull Request → CI → Review → Merge to dev
```

Required checks must pass before integration.

# 3. CI Validation
Initial required checks:

* Backend build;
* Frontend build;
* unit tests;
* important authorization tests;
* important API tests.
  Progressive checks may include:
* secret scanning;
* dependency scanning;
* static analysis;
* container scanning.
  Blocking and advisory checks must be clearly distinguished.

# 4. Artifact Model
Frontend and Backend are separate images published to GHCR.
Each artifact shall be traceable to:

* source commit;
* GitHub Actions run;
* application version;
* test/scan results.
  Promotion uses an immutable identity, preferably an image digest.
  `latest` is not an authoritative deployment identity.

# 5. Configuration and Secrets
```text
Same Image
├── DEV config + secrets
├── STAGE config + secrets
└── PROD config + secrets
```

Environment configuration and secrets remain outside images.
Secrets shall not appear in source, Dockerfiles, Frontend bundles, images, or CI logs.

# 6. Promotion Flow
```text
Merge to dev
→ Build/Test/Scan
→ Publish GHCR
→ Deploy DEV
→ Validate DEV
→ Promote same artifact to STAGE
→ Validate STAGE
→ Manual PROD approval
→ PROD
```

DEV deployment becomes automatic when the deployment phase is active.
Failed DEV or STAGE validation stops promotion.
PROD requires explicit approval.

# 7. Deployment Verification
Verify:

* expected version and commit;
* container/Pod health;
* Backend readiness;
* Flyway result;
* absence of restart loops/startup failures;
* basic authenticated smoke test.
  Detailed telemetry belongs to `observability-design.md`.

# 8. Smoke Tests
Smoke tests should verify:

* correct version is running;
* authentication works;
* Backend reaches PostgreSQL;
* one important API workflow succeeds;
* critical dependencies are usable.
  Smoke tests do not replace full automated tests.

# 9. Database Migrations
Flyway uses the same ordered migration history across DEV, STAGE, and PROD.
Potentially destructive changes should use:

```text
Expand → Migrate → Contract
```

where needed for deployment and rollback compatibility.
Only one migration process may modify a schema at a time.
Exact migration execution remains open.

# 10. Rollback
PROD shall support redeployment of a previous verified immutable artifact.

```text
Current Artifact → failure → Previous Stable Artifact → Redeploy + Verify
```

Rollback requires compatible configuration and database schema.
`Git revert` and artifact rollback are separate operations.
Database recovery belongs to `backup-recovery.md`.

# 11. Pipeline Security
CI/CD follows least privilege:

* PR workflows receive no PROD credentials;
* deployment authority is environment-specific;
* PROD approval is protected;
* GHCR publication is controlled;
* deployment identities are restricted;
* third-party Actions are reviewed;
* long-lived secrets are minimized.
  Exact credential technology remains open.

# 12. Failure Rules
| Failure                                       | Result                        |
| --------------------------------------------- | ----------------------------- |
| Build/test failure                            | Stop                          |
| Mandatory security gate failure               | Stop                          |
| Image publication failure                     | Stop deployment               |
| DEV validation failure                        | Stop STAGE promotion          |
| STAGE validation failure                      | Stop PROD promotion           |
| Flyway failure                                | Stop deployment               |
| PROD deployment failure                       | Investigate / rollback        |
| Registry unavailable                          | Existing deployment continues |
| Unverified artifacts shall never be promoted. |                               |

# 13. Responsibility Split
```text
Terraform → infrastructure lifecycle
Ansible   → host/server configuration
CI/CD     → application build and deployment
```

Terraform and Ansible do not run for every normal application deployment.

# 14. PROD Kubernetes
Before GitOps, GitHub Actions may deploy the approved artifact directly to Kubernetes using a restricted deployment identity.

```text
GitHub Actions → Kubernetes API → Deployment → Pods
```

CI does not SSH to individual Kubernetes nodes.

# 15. GitOps Progression
```text
Manual Deployment → CI/CD → Direct Kubernetes → GitOps → Argo CD
```

GitOps follows direct Kubernetes deployment.
Future flow:

```text
Application Repo → GitHub Actions → Build/Test/Scan → GHCR
GitOps Repo → Argo CD → Kubernetes
```

The GitOps repository references an already-built immutable image.
Argo CD runs inside Kubernetes.
Manual sync may be used initially.

# 16. Traceability
```text
Running Container
→ Image Digest
→ GHCR Artifact
→ GitHub Actions Run
→ Git Commit
→ Pull Request
```

# 17. Fixed Rules
1. GitHub is the collaboration platform.
2. GitHub Actions is the CI/CD platform.
3. GHCR is the container registry.
4. `dev` is the integration branch.
5. `main` contains stable/completed states.
6. Changes are integrated through Pull Requests.
7. Required checks must pass before integration.
8. Frontend and Backend are separate images.
9. Images are built once and promoted unchanged.
10. Configuration and secrets remain external.
11. DEV becomes automatic after successful integration CI.
12. STAGE receives the artifact validated in DEV.
13. PROD requires explicit manual approval.
14. Failed DEV/STAGE validation stops promotion.
15. Rollback uses a previous immutable artifact.
16. Flyway migrations must consider rollback compatibility.
17. Terraform and Ansible are separate from normal app delivery.
18. Direct Kubernetes deployment precedes GitOps.
19. Argo CD follows direct Kubernetes deployment.
20. Git defines desired PROD state in the GitOps phase.

# 18. Open Decisions
1. Branch-protection rules.
2. Mandatory checks by project phase.
3. GHCR naming and versioning convention.
4. STAGE promotion trigger.
5. PROD approver policy.
6. Deployment credential mechanism.
7. Flyway execution mechanism.
8. Smoke-test suite.
9. Artifact/evidence retention.
10. Blocking vulnerability severities.
11. Point when GitOps replaces direct PROD deployment.
12. Whether Argo CD auto-sync is enabled later.