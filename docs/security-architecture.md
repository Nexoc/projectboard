# ProjectBoard Security Architecture
| Field | Value |
| --- | --- |
| Status | Working architecture baseline |
| Parent | [High-Level Design](hld.md) |
| Network | [Network Design](network-design.md) |
| Last updated | 2026-09-03 |

This document defines Version 1 security boundaries and high-level controls.
Detailed firewall rules, IPs, credentials, and product configuration belong to lower-level designs and Infrastructure as Code.

# 1. Security Principles
ProjectBoard follows:
1. authenticate every protected request;
2. authorize every protected business action in the Backend;
3. deny when identity, ownership, or permission cannot be established safely;
4. isolate DEV, STAGE, and PROD;
5. expose only required public endpoints;
6. keep secrets outside Git, source, images, browser code, and logs;
7. use least privilege and separate human/workload identities;
8. record security-relevant events;
9. never treat internal network location as sufficient trust.

# 2. Trust Boundaries
Primary trust zones are the public Internet, Gateway, DEV/STAGE/PROD, Security VM, Storage VM, Monitoring VM, management plane, and future AI/Ollama boundary.
Crossing a trust boundary requires an approved network path and appropriate identity/authorization.
Exact network flows belong to `network-design.md`.

# 3. Authentication
Keycloak owns user identity, login/logout, credential verification, token issuance, and OIDC configuration.
ProjectBoard stores no passwords and does not implement a second authentication system.

```text
Browser
  ↓ HTTPS
Gateway
  ↓
Keycloak public OIDC endpoints
````

The Security VM itself is not generally public.

# 4. Browser Client
Angular is a public browser client and shall not contain a confidential client secret.
Environment-specific OIDC configuration may contain issuer, client ID, redirect URIs, logout redirects, and allowed origins.
Exact browser OIDC flow, token storage, refresh, and logout behavior remain open.

# 5. Backend Token Validation
The Spring Boot Backend validates protected-request tokens, including applicable signature, issuer, expiration, claims, and API-context checks.
Invalid, malformed, expired, or untrusted tokens are rejected.
A valid Keycloak token proves identity only; it does not grant access to every Project.

# 6. Business Authorization
ProjectBoard owns business authorization.

```text
Identity + Project Membership + Project Role + Resource Ownership + Action
```

Version 1 roles are `PROJECT_OWNER` and `MEMBER`.
There is no global ProjectBoard `ADMIN` business role.
Infrastructure administrators do not automatically receive ProjectBoard business permissions.

# 7. Project Isolation
Every protected resource operation resolves its authoritative owning Project.
Changing a URL, ID, parameter, or request body must not bypass Project isolation.
A resource identifier is a locator, not an authorization capability.
If identity, membership, ownership, or permission cannot be established safely, access is denied.

# 8. Environment Isolation
DEV, STAGE, and PROD use separate business data, credentials, secrets, configuration, Redis state, RabbitMQ state, MinIO scopes, and identity-provider configuration.
Lower environments shall not access PROD data, secrets, or messaging state.
Shared infrastructure does not imply shared application permissions.

# 9. Security VM
The accepted shared Security VM hosts:

```text
security
├── Keycloak
├── Keycloak DB
└── Vault
```

It is a restricted infrastructure host, not a general application server.
Public Keycloak authentication and Keycloak administration use separate trust paths.
Keycloak administration is available only through the controlled management path.

# 10. Secrets and Vault
Secrets include database/broker credentials, private keys, Vault tokens, MinIO secrets, and other confidential service credentials.
Secrets shall not appear in Git, source code, Pull Requests, application images, Frontend bundles, ordinary logs, or documentation containing real operational values.
ProjectBoard follows `Artifact ≠ Configuration ≠ Secrets`.

Vault is the target centralized secrets manager with DEV/STAGE/PROD separation and least-privilege policies.
Before full Vault integration, external secret injection is acceptable only when outside Git/images, environment-specific, access-controlled, and excluded from logs.
Vault bootstrap, recovery, workload authentication, leases, rotation, and break-glass remain open.

# 11. Workload Identities
Human and software identities are separate.

Examples:

* Backend → PostgreSQL: environment-specific runtime identity;
* Backend → Redis: environment-specific workload access;
* Publisher/Consumer → RabbitMQ: environment-specific broker identity;
* Backend → MinIO: environment-specific object identity;
* Workload → Vault: environment-specific Vault identity/policy;
* CI/CD → target: restricted deployment identity.

PROD credentials shall not be usable by DEV or STAGE.

# 12. Internal Services
The following shall not be publicly exposed: PostgreSQL, Redis, RabbitMQ, Vault, MinIO administration, monitoring administration, Kubernetes API, future AI Service, and Ollama.
Detailed restrictions belong to `network-design.md`.

# 13. Gateway and TLS
Nginx is the controlled public entry point and provides TLS termination, HTTPS enforcement, reverse proxying, hostname routing, basic rate limiting, and public Keycloak OIDC routing.
Public application and authentication traffic uses HTTPS.
TCP 80 may exist only for redirect or certificate automation where required.
Universal internal mTLS is not required initially; exact internal TLS/PKI boundaries remain open.

# 14. Rate Limiting
Basic rate limiting is part of Version 1.
Initial targets may include authentication traffic, sensitive API operations, and repeated failed requests.
Nginx provides generic edge limiting; Redis/application-level limiting may be added for distributed or business-aware limits.
Rate limiting does not replace authentication, authorization, or validation.

# 15. Administrative Access
Administrative interfaces use a controlled management path, including SSH, Keycloak Admin, Vault Admin, Grafana, PostgreSQL Admin, MinIO Admin, and Kubernetes API.
The exact mechanism may use VPN, private management network, bastion, or another controlled solution.
Named human identities, least privilege, and authentication logging are preferred.
Detailed SSH hardening belongs to infrastructure configuration.

# 16. Logging and Security Events
ProjectBoard distinguishes:

| Type           | Purpose                             | Primary Store          |
| -------------- | ----------------------------------- | ---------------------- |
| Business Audit | Who changed business state?         | PostgreSQL             |
| Security Event | Was access suspicious or denied?    | Security observability |
| Technical Log  | Why did the system behave this way? | Loki                   |
| Keycloak Event | What happened in authentication?    | Keycloak / forwarding  |

Security events include failed authentication, invalid tokens, authorization denials, repeated 401/403, rate-limit violations, Vault failures, and SSH authentication failures.
Logs shall not intentionally contain passwords, access/refresh tokens, Authorization headers, Vault tokens, database/broker passwords, MinIO secret keys, or private keys.
Correlation IDs may connect evidence across systems without merging these record types.

# 17. Monitoring and Backup Security
Monitoring receives only required telemetry paths and does not automatically gain shell, database-admin, or Vault-admin access.
Backups require restricted identities, non-public storage, protected transfer, and controlled restore access.
Detailed controls belong to `observability-design.md` and `backup-recovery.md`.

# 18. Failure Behavior
Security failures fail safely:

* invalid authentication → reject;
* uncertain authorization → deny;
* Keycloak failure → never broaden authorization;
* Vault failure → no fallback to hard-coded credentials;
* Redis failure → temporary features may degrade, authorization remains enforced;
* RabbitMQ failure → messaging may delay, authorization is unaffected;
* monitoring failure → core business processing continues.

# 19. Future Kubernetes and AI
When PROD Kubernetes is activated, security expands to RBAC, service accounts, NetworkPolicies, workload controls, restricted ingress/egress, secret delivery, and restricted API access.
AI is outside Version 1.

Prohibited future paths:

```text
Frontend ─X─► AI Service
AI Service ─X─► ProjectBoard PostgreSQL
Public Internet ─X─► Ollama
```

AI output is untrusted advisory input and requires user review, Backend validation, and authorization before becoming business data.

# 20. Security Verification
Important controls shall be testable:

* unauthenticated and invalid-token requests are rejected;
* non-members cannot access Projects;
* MEMBER cannot perform owner-only operations;
* changed resource IDs cannot bypass authorization;
* lower environments cannot access PROD secrets/data;
* internal services are not publicly reachable;
* Keycloak Admin is not exposed through the public login path;
* browser code contains no confidential client secret;
* repository and logs do not expose secrets;
* rate-limit violations are observable.

# 21. Fixed Rules
1. Keycloak owns authentication.
2. Backend owns business authorization.
3. ProjectBoard stores no user passwords.
4. Protected operations are authorized server-side.
5. Resource IDs never grant access by themselves.
6. Authorization uncertainty results in denial.
7. No global ProjectBoard `ADMIN` exists in Version 1.
8. DEV, STAGE, and PROD use separate data and credentials.
9. Secrets are absent from Git, images, browser code, and logs.
10. Vault is the centralized target for secrets.
11. PostgreSQL, Redis, RabbitMQ, and infrastructure admin interfaces are internal.
12. Public application and OIDC traffic enter through the Gateway using HTTPS.
13. Keycloak authentication and administration use separate trust paths.
14. Basic rate limiting is part of Version 1.
15. Human and workload identities are separate.
16. Administrative access uses a controlled management path.
17. Business Audit, Security Events, and Technical Logs remain distinct.
18. Monitoring access does not imply unrestricted administration.
19. Future AI cannot directly modify ProjectBoard business data.
20. Ollama is never publicly exposed.

# 22. Open Decisions
1. Keycloak realm/client topology, claim mapping, and token audience.
2. Token lifetime, renewal, browser storage, and OIDC browser flow.
3. Frontend CSP and detailed XSS controls.
4. Management-access mechanism, MFA, and break-glass.
5. Vault bootstrap, recovery, workload authentication, and rotation.
6. Internal TLS boundaries and PKI.
7. Security-event retention and tamper-resistant storage if required.
8. MinIO object-access strategy.
9. Nginx versus application/Redis rate-limit responsibilities.
10. Kubernetes workload identity.
11. CI/CD deployment identity and mandatory security gates.