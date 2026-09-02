# Software Requirements Specification

## ProjectBoard DevSecOps Lab

---

# 1. Purpose

## 1.1 System Overview

ProjectBoard is a web-based project management application for small software development teams.

The application combines three basic concepts:

* Product Backlog
* Sprint Planning
* Kanban Board

The system is intentionally designed with relatively simple business logic.

It is **not intended to replace Jira, Azure DevOps, YouTrack or similar commercial project management platforms**.

The main purpose is to build a realistic full-stack application that can later be used as a practical environment for learning and experimenting with:

* backend development
* frontend development
* REST APIs
* authentication and authorization
* databases
* containerization
* networking
* security
* CI/CD
* monitoring
* logging
* backup and restore
* deployment
* Kubernetes
* DevSecOps
* local AI integration in a future version

The core principle of the project is:

```text
Business Logic = simple
Infrastructure = progressively complex
```

---

## 1.2 Business Purpose

ProjectBoard shall allow a small development team to organize project work without unnecessary complexity.

Users shall be able to:

* create projects
* invite registered users to projects
* maintain a backlog
* create User Stories
* break User Stories into Tasks
* create Sprints
* select User Stories for a Sprint
* view the current Sprint
* assign Tasks to project members
* organize Sprint work on a Kanban Board
* move User Stories through workflow columns
* complete Tasks

---

## 1.3 Educational Purpose

ProjectBoard also serves as a DevSecOps learning project.

The project shall provide a realistic application around which infrastructure can progressively be developed.

The application should make it possible to practice:

```text
Git
GitHub
Linux
Java
Spring Boot
Angular
PostgreSQL
REST
OpenAPI

Docker
Docker Compose
Docker Networking
Nginx

Keycloak
OAuth2
OpenID Connect
JWT
RBAC

TLS
Secrets Management
Vault

Redis
MinIO

GitHub Actions
CI/CD
Security Scanning
Container Registry

Prometheus
Grafana
Loki
Grafana Alloy
Alertmanager

Backup
Restore

Kubernetes
Kubernetes Security

Local AI later
```

The infrastructure shall evolve without requiring unnecessary growth of the business domain.

---

## 1.4 Scope of Version 1

Version 1 shall focus on the following business workflow:

```text
Project
   │
   ├── Members
   │
   ├── Backlog
   │     └── User Stories
   │           └── Tasks
   │
   ├── Sprints
   │     └── selected User Stories
   │
   └── Kanban Board
         └── Columns
               └── User Stories
```

A separate `Card` entity shall **not** exist.

A User Story is the item displayed as a card on the Kanban Board.

This avoids having overlapping concepts such as:

```text
UserStory
Card
Task
```

The simplified model is:

```text
UserStory
└── Task
```

---

# 2. Users and Roles

ProjectBoard shall use two project-level roles in Version 1:

```text
PROJECT_OWNER
MEMBER
```

There shall be no application-level `ADMIN` functionality in Version 1.

---

## 2.1 Authenticated User

A user must authenticate before accessing protected ProjectBoard functionality.

Authentication shall be delegated to an external Identity Provider.

For this project, the Identity Provider shall be Keycloak.

ProjectBoard shall not manage or store user passwords itself.

A user account shall have a stable external identity that can be referenced by ProjectBoard.

---

## 2.2 Project Owner

The user who creates a project shall automatically become the:

```text
PROJECT_OWNER
```

A Project Owner shall be able to:

* view the project
* update project information
* delete the project
* view available registered users
* add registered users to the project
* remove members from the project
* perform normal project work

The Project Owner shall also have all permissions available to a Member.

A Project Owner shall not be able to accidentally remove themselves from their own project in Version 1.

Ownership transfer is not required in Version 1.

---

## 2.3 Member

A user added to a project becomes a:

```text
MEMBER
```

A Member shall be able to work with project content.

This includes:

* User Stories
* Tasks
* Sprints
* Kanban Columns
* Kanban workflow

Members shall not be allowed to:

* delete the project
* modify project membership
* change project ownership

---

## 2.4 Project Access Rule

Project information shall only be accessible to users who belong to that project.

The following must never be possible:

```text
User A
    ↓
Project B
    ↓
User A is not Owner or Member
    ↓
ACCESS DENIED
```

Authorization must be checked on the server side.

Frontend restrictions alone shall not be considered sufficient authorization.

---

# 3. Functional Requirements

# 3.1 Authentication

## FR-AUTH-01 — Login

The system shall allow a registered user to authenticate through the configured Identity Provider.

After successful authentication, the user shall be able to access authorized ProjectBoard functionality.

---

## FR-AUTH-02 — Registration

User registration may be provided through the configured Identity Provider.

ProjectBoard itself shall not implement password storage or password verification.

---

## FR-AUTH-03 — Logout

The authenticated user shall be able to terminate their authenticated session.

---

## FR-AUTH-04 — Unauthorized Access

The system shall reject requests to protected functionality when the user is not authenticated.

---

# 3.2 Project Management

## FR-PROJECT-01 — Create Project

An authenticated user shall be able to create a new project.

At minimum, a project shall contain:

* name
* optional description

The creator shall automatically become the Project Owner.

---

## FR-PROJECT-02 — List Projects

A user shall be able to view projects in which they participate as:

* Project Owner
* Member

Projects to which the user does not belong shall not be displayed.

---

## FR-PROJECT-03 — View Project

A project member shall be able to view the project and its permitted project data.

---

## FR-PROJECT-04 — Update Project

Only the Project Owner shall be able to update project-level information.

---

## FR-PROJECT-05 — Delete Project

Only the Project Owner shall be able to delete the project.

Project deletion shall require explicit confirmation.

Deleting the project shall also remove or invalidate its project-specific data according to the defined persistence rules.

---

# 3.3 Project Membership

## FR-MEMBER-01 — View Available Users

The Project Owner shall be able to view registered users that can be added to the project.

Only information necessary for identifying and selecting a user shall be shown.

---

## FR-MEMBER-02 — Add Member

The Project Owner shall be able to add a registered user to the project.

The added user shall receive the project role:

```text
MEMBER
```

---

## FR-MEMBER-03 — Remove Member

The Project Owner shall be able to remove a Member from the project.

After removal, that user shall no longer be authorized to access protected project information.

---

## FR-MEMBER-04 — Prevent Duplicate Membership

The same user shall not be added to the same project more than once.

---

# 3.4 Backlog

## FR-BACKLOG-01 — Project Backlog

Every project shall have a Backlog.

The Backlog represents User Stories that are available for planning.

---

## FR-BACKLOG-02 — View Backlog

Project members shall be able to view User Stories available in the project Backlog.

---

## FR-BACKLOG-03 — Sprint Selection

Project members shall be able to select User Stories from the Backlog and associate them with a Sprint.

---

# 3.5 User Stories

## FR-STORY-01 — Create User Story

A project member shall be able to create a User Story.

A User Story shall contain at minimum:

* title

It may additionally contain:

* description

---

## FR-STORY-02 — View User Story

A project member shall be able to view the details of a User Story.

---

## FR-STORY-03 — Update User Story

A project member shall be able to update a User Story.

---

## FR-STORY-04 — Delete User Story

A project member shall be able to delete a User Story.

Deletion shall require confirmation when associated child data such as Tasks would also be affected.

---

## FR-STORY-05 — Story Belongs to Project

Every User Story shall belong to exactly one Project.

A User Story shall not be visible or editable outside its Project.

---

# 3.6 Tasks

## FR-TASK-01 — Create Task

A project member shall be able to create a Task inside a User Story.

A Task shall contain at minimum:

* title

It may contain:

* description
* assignee
* completion state

---

## FR-TASK-02 — Update Task

A project member shall be able to modify an existing Task.

---

## FR-TASK-03 — Delete Task

A project member shall be able to delete a Task.

---

## FR-TASK-04 — Complete Task

A project member shall be able to mark a Task as completed.

The user shall also be able to mark it as incomplete again if necessary.

---

## FR-TASK-05 — Assign Task

A project member shall be able to assign a Task to a project Member.

To keep Version 1 simple, a Task shall have at most:

```text
one assignee
```

A Task may also remain unassigned.

---

## FR-TASK-06 — Assignment Validation

Only users who currently belong to the Project shall be assignable to a Task.

---

## FR-TASK-07 — Task Ownership

Every Task shall belong to exactly one User Story.

A Task shall therefore indirectly belong to exactly one Project.

---

# 3.7 Sprint Management

## FR-SPRINT-01 — Create Sprint

A project member shall be able to create a Sprint.

A Sprint shall contain at minimum:

* name
* start date
* end date

---

## FR-SPRINT-02 — Date Validation

A Sprint end date shall not be earlier than its start date.

---

## FR-SPRINT-03 — Update Sprint

A project member shall be able to update Sprint information.

---

## FR-SPRINT-04 — Delete Sprint

A project member shall be able to delete a Sprint.

The system shall prevent accidental deletion of Sprint planning data through appropriate confirmation.

---

## FR-SPRINT-05 — Add User Story to Sprint

A project member shall be able to add a User Story to a Sprint.

Sprint planning shall occur at the User Story level.

Tasks shall remain children of the selected User Story.

The system shall therefore use:

```text
Sprint
└── User Stories
      └── Tasks
```

and not:

```text
Sprint
└── independently selected Tasks
```

---

## FR-SPRINT-06 — Remove User Story from Sprint

A project member shall be able to remove a User Story from a Sprint.

The User Story shall then remain available in the Project Backlog.

---

## FR-SPRINT-07 — Current Sprint

The system shall allow one Sprint to be identified as the current Sprint.

A project shall have at most:

```text
one Current Sprint
```

at a time.

---

## FR-SPRINT-08 — View Current Sprint

Project members shall be able to view the Current Sprint.

The Current Sprint view shall show at least:

* sprint name
* start date
* end date
* selected User Stories

---

## FR-SPRINT-09 — No Automatic Sprint Scheduling

Version 1 shall not automatically start or complete Sprints based only on dates.

Sprint state shall remain under explicit user control.

This avoids hidden business logic and simplifies the first implementation.

---

# 3.8 Kanban Board

## FR-BOARD-01 — Project Board

Each Project shall have a Kanban Board.

The Board shall organize User Stories using columns.

---

## FR-BOARD-02 — User Story as Kanban Card

The visual card displayed on the Kanban Board shall represent a User Story.

No separate business entity named `Card` shall exist.

---

## FR-BOARD-03 — Current Sprint Board

The primary Kanban Board view shall display User Stories associated with the Current Sprint.

This provides the workflow:

```text
Backlog
   ↓
Sprint Planning
   ↓
Current Sprint
   ↓
Kanban Board
```

---

# 3.9 Board Columns

## FR-COLUMN-01 — Default Columns

A newly created Project should initially provide basic workflow columns:

```text
To Do
In Progress
Done
```

---

## FR-COLUMN-02 — Create Column

Project members shall be able to create additional Kanban Columns.

New columns shall be added to the board workflow.

---

## FR-COLUMN-03 — Update Column

Project members shall be able to rename a Kanban Column.

---

## FR-COLUMN-04 — Delete Column

Project members shall be able to delete an empty Kanban Column.

A Column containing User Stories shall not be deleted until those User Stories have been moved elsewhere.

---

## FR-COLUMN-05 — Move User Story

Project members shall be able to move a User Story from one Kanban Column to another.

The new position shall be persisted.

---

# 3.10 API

## FR-API-01 — REST API

Core ProjectBoard functionality shall be accessible through a REST API.

---

## FR-API-02 — API Contract

The REST API shall be described using OpenAPI.

The specification shall be stored in:

```text
api/openapi.yaml
```

---

## FR-API-03 — Consistent Error Responses

The API shall return consistent error responses for at least:

* invalid input
* unauthenticated access
* unauthorized access
* resource not found
* business rule violation
* internal server error

---

## FR-API-04 — Input Validation

The backend shall validate incoming data before processing or persisting it.

Frontend validation shall not replace backend validation.

---

# 4. Non-Functional Requirements

# 4.1 Security

## NFR-SEC-01 — Authentication

Protected functionality shall require authentication.

---

## NFR-SEC-02 — Authorization

Authorization shall be enforced on the backend.

---

## NFR-SEC-03 — Project Isolation

A user shall not be able to access another Project by manipulating:

* URLs
* API identifiers
* request parameters
* request bodies

unless the user is an authorized Project Member.

---

## NFR-SEC-04 — Password Handling

ProjectBoard shall not store user passwords.

Authentication credentials shall be handled by the configured Identity Provider.

---

## NFR-SEC-05 — Secrets

Secrets shall not be stored in source code.

This includes:

* database credentials
* client secrets
* API keys
* encryption keys

---

## NFR-SEC-06 — TLS

External production-like communication shall use HTTPS/TLS.

---

## NFR-SEC-07 — Secure Defaults

Infrastructure services that do not require public access shall not be directly exposed to the Internet.

Examples include:

```text
PostgreSQL
Redis
Vault
Prometheus
Loki
Ollama
```

---

# 4.2 Data Integrity

## NFR-DATA-01 — Persistent Data

Application data shall survive application and container restarts.

---

## NFR-DATA-02 — Database Migrations

Database schema changes shall be reproducible and version controlled.

---

## NFR-DATA-03 — Referential Integrity

The system shall prevent invalid relationships such as:

* Task assigned to a non-member
* Task belonging to a User Story from another Project
* User Story associated with an unrelated Sprint
* duplicate Project Membership

---

# 4.3 Maintainability

## NFR-MAIN-01 — Separation of Responsibilities

The project shall maintain a clear separation between:

```text
Frontend
Backend
API Contract
Infrastructure
Documentation
Future AI Service
```

---

## NFR-MAIN-02 — Replaceable Infrastructure

Infrastructure components should be replaceable without rewriting the core business logic.

---

## NFR-MAIN-03 — Documentation

Important requirements, architecture decisions, security decisions and deployment procedures shall be documented.

---

# 4.4 Testability

## NFR-TEST-01 — Automated Tests

Important business rules shall have automated tests.

---

## NFR-TEST-02 — Authorization Tests

Security-sensitive access rules shall have automated tests.

Examples:

```text
Member can access own Project
Non-member cannot access Project
Member cannot manage Project membership
Project Owner can manage Project membership
```

---

## NFR-TEST-03 — API Tests

Important REST endpoints shall be testable independently of the frontend.

---

# 4.5 Containerization

## NFR-CONTAINER-01

The application shall be containerizable.

---

## NFR-CONTAINER-02

The development/runtime environment shall be reproducible using Docker Compose before Kubernetes is introduced.

---

## NFR-CONTAINER-03

Containers should run with the minimum privileges required.

---

## NFR-CONTAINER-04

Containers shall provide health information where technically appropriate.

---

# 4.6 Networking

## NFR-NET-01

Application and infrastructure components shall be separated using appropriate internal networks.

---

## NFR-NET-02

Database and infrastructure services shall not require public Internet exposure.

---

## NFR-NET-03

External HTTP/HTTPS traffic shall enter the application through a controlled reverse proxy.

---

# 4.7 CI/CD and DevSecOps

## NFR-CI-01

Pull Requests shall trigger automated validation.

---

## NFR-CI-02

The CI pipeline shall build the Backend and Frontend.

---

## NFR-CI-03

The CI pipeline shall execute automated tests.

---

## NFR-CI-04

The project should integrate automated security checks.

Examples include:

* dependency scanning
* SAST
* secret scanning
* container image scanning

---

## NFR-CI-05

A failed mandatory CI check shall prevent a change from being considered ready for merge.

---

# 4.8 Observability

## NFR-OBS-01 — Metrics

The system should expose operational metrics for:

* application availability
* HTTP requests
* response time
* HTTP errors
* JVM
* CPU
* memory
* disk
* container health

---

## NFR-OBS-02 — Logging

Application and infrastructure logs should be centrally collectable.

---

## NFR-OBS-03 — Log Search

Collected logs should be searchable through a centralized interface.

---

## NFR-OBS-04 — Alerting

The infrastructure should support alerts for conditions such as:

* Backend unavailable
* Database unavailable
* high CPU
* high memory
* low disk space
* excessive HTTP 500 responses

---

# 4.9 Backup and Recovery

## NFR-BACKUP-01

Persistent project data shall be backupable.

---

## NFR-BACKUP-02

The backup process shall be documented.

---

## NFR-BACKUP-03

A backup shall not be considered successfully implemented until restoration has been tested.

The project rule is:

```text
Backup without Restore Test is not complete.
```

---

# 4.10 Deployment

## NFR-DEPLOY-01

The application shall be deployable to the project's Linux server environment.

---

## NFR-DEPLOY-02

Deployment shall be reproducible from repository configuration and documentation.

---

## NFR-DEPLOY-03

The first deployment model may use Docker Compose.

---

## NFR-DEPLOY-04

The architecture shall later support migration to Kubernetes.

---

# 4.11 Performance

Version 1 does not target large-scale workloads.

However:

## NFR-PERF-01

Normal user operations should respond without unnecessary blocking under expected small-team usage.

---

## NFR-PERF-02

Database queries shall avoid obvious unnecessary repeated access patterns.

---

## NFR-PERF-03

Long-running future AI operations shall not be allowed to block normal ProjectBoard operations.

---

# 4.12 Usability

## NFR-UX-01

The user interface shall prioritize clarity over visual complexity.

---

## NFR-UX-02

The primary workflow shall remain understandable:

```text
Project
→ Backlog
→ User Story
→ Sprint
→ Kanban Board
→ Done
```

---

## NFR-UX-03

Version 1 shall use English as the primary interface language.

Additional languages are optional future functionality.

---

# 4.13 Educational Requirements

Because ProjectBoard is also a learning project:

## NFR-EDU-01

Every major infrastructure technology shall be introduced only after its purpose is understood.

For each major technology, the project should answer:

```text
1. What problem does it solve?
2. How was that problem handled before introducing it?
3. How does the technology work?
4. How can we verify that it works correctly?
```

---

## NFR-EDU-02

Technologies shall not be added solely to increase the number of containers or tools.

---

## NFR-EDU-03

The developer shall be able to explain important architectural and implementation decisions.

---

# 5. Out of Scope

The following functionality is explicitly excluded from Version 1.

## 5.1 Retrospectives

Version 1 shall not contain:

* Sprint Retrospective
* positive retrospective feedback
* negative retrospective feedback
* improvement actions

Retrospective functionality may be considered only after the core application and infrastructure are stable.

---

## 5.2 AI

Version 1 shall not contain:

* AI-generated Tasks
* AI-generated tickets
* LLM integration
* Vision Model integration
* screenshot analysis
* autonomous AI actions

AI is planned as a separate future project phase.

The intended future workflow is:

```text
User Story
    +
Screenshot / Image
        ↓
AI Service
        ↓
Local LLM / Vision
        ↓
Suggested Tasks / Tickets
        ↓
Human Review
        ↓
Accept / Reject
        ↓
Project Tasks
```

The AI shall not automatically modify project business data without user confirmation.

---

## 5.3 Notifications

Version 1 shall not include:

* email notifications
* push notifications
* automatic notifications when work reaches Done

---

## 5.4 Realtime Collaboration

Version 1 shall not include:

* WebSockets
* realtime board synchronization
* realtime collaborative editing

Normal HTTP refresh/request behavior is sufficient.

---

## 5.5 Chat

The application shall not provide an integrated chat system in Version 1.

---

## 5.6 General File Attachments

General attachments to:

* Projects
* User Stories
* Tasks

are not required in Version 1.

Object storage may still be introduced later as an infrastructure learning component and for future AI image processing.

---

## 5.7 Global Administration

Version 1 shall not contain:

```text
ADMIN
SUPER_ADMIN
global user management
global role management
```

User identity management remains the responsibility of the Identity Provider.

---

## 5.8 Ownership Transfer

Transfer of Project ownership from one user to another is not required in Version 1.

---

## 5.9 Advanced Permissions

Version 1 shall not implement complex permission structures such as:

* custom roles
* per-Task permissions
* per-User-Story permissions
* permission inheritance
* organization-level permissions

Only:

```text
PROJECT_OWNER
MEMBER
```

are required.

---

## 5.10 Mobile Applications

Native Android and iOS applications are outside the scope.

---

## 5.11 Advanced Sprint Management

Version 1 shall not require:

* automatic Sprint start
* automatic Sprint completion
* velocity calculation
* burndown charts
* story points
* capacity planning
* Sprint analytics

---

## 5.12 Advanced Project Management

Version 1 shall not require:

* dependencies between Tasks
* dependencies between User Stories
* subtasks below Task level
* epics
* roadmaps
* milestones
* labels
* custom fields
* time tracking

---

# 6. MoSCoW Requirements

# Must

The following functionality is required for the first complete usable version.

### Authentication

* login through Keycloak / OIDC
* authenticated access
* logout
* backend authorization

### Projects

* create Project
* list accessible Projects
* view Project
* update Project as Project Owner
* delete Project as Project Owner

### Membership

* automatically assign creator as Project Owner
* add registered Member
* remove Member
* protect Projects from non-members

### User Stories

* create
* view
* update
* delete

### Tasks

* create Task inside User Story
* update Task
* delete Task
* complete/uncomplete Task
* assign one Project Member to Task

### Backlog

* view Project Backlog
* use User Stories for Sprint planning

### Sprints

* create Sprint
* update Sprint
* delete Sprint
* select User Stories for Sprint
* remove User Stories from Sprint
* identify one Current Sprint
* view Current Sprint

### Kanban

* Board
* default workflow columns
* create Column
* rename Column
* delete empty Column
* move User Stories between Columns

### API

* REST API
* OpenAPI contract
* validation
* consistent error responses

### Persistence

* PostgreSQL persistence
* database migrations

### Security

* authenticated access
* project-level authorization
* no secrets in source repository

### Development

* Git workflow
* Pull Requests
* automated backend build
* automated frontend build
* automated tests

---

# Should

These requirements are important for the DevSecOps objective and should be implemented after the core application works.

* Docker
* Docker Compose
* Nginx reverse proxy
* Docker network isolation
* HTTPS / TLS
* Redis
* Vault
* CI/CD pipeline
* container registry
* dependency scanning
* SAST
* secret scanning
* container image scanning
* Prometheus
* Grafana
* Loki
* Grafana Alloy
* Alertmanager
* centralized logging
* backup
* restore testing
* container hardening
* DEV / STAGE separation

---

# Could

These features may be implemented if time permits or during later infrastructure phases.

* German/Russian interface
* additional interface languages
* MinIO
* additional monitoring dashboards
* advanced rate limiting
* richer health checks
* more advanced audit information
* Kubernetes deployment
* Kubernetes NetworkPolicies
* Kubernetes RBAC hardening
* GitOps
* Argo CD or Flux
* additional development environments
* additional security hardening

---

# Won't

The following are explicitly excluded from Version 1:

* Retrospective
* retrospective feedback
* improvement actions
* AI
* local LLM
* Vision Model
* automatic ticket generation
* autonomous AI modification of Project data
* notifications
* chat
* realtime updates
* WebSockets
* general file attachments
* ADMIN role
* ownership transfer
* complex permissions
* native mobile applications
* story points
* velocity
* burndown charts
* advanced Sprint analytics
* epics
* roadmaps
* time tracking
* high availability
* large-scale distributed production architecture

These items may be reconsidered only after the first version and the planned DevSecOps infrastructure are stable.

---

# 7. Definition of Done

The Definition of Done applies at three levels:

```text
Feature
Project Phase
Version 1
```

---

## 7.1 Feature Definition of Done

A feature is considered Done only when all applicable conditions are satisfied.

### Requirement

* The feature corresponds to a defined requirement.
* Expected behavior is clear.
* Relevant edge cases are understood.

### Implementation

* Required backend functionality is implemented.
* Required frontend functionality is implemented.
* Required persistence changes are implemented.
* Database schema changes use version-controlled migrations.

### API

* Relevant REST endpoints are implemented.
* OpenAPI documentation is updated.
* Request validation is implemented.
* Error behavior is defined.

### Security

* Authentication requirements are respected.
* Authorization is enforced on the backend.
* Project isolation is preserved.
* No credentials or secrets are committed to Git.

### Testing

* Important business logic has automated tests.
* Authorization-sensitive functionality has security tests.
* Existing tests continue to pass.
* Relevant failure cases are tested.

### CI

* Backend build succeeds.
* Frontend build succeeds.
* Required automated tests pass.
* Mandatory CI checks are green.

### Runtime

* The feature works in the supported local development environment.
* The feature works in the Docker-based environment when containerization applies.
* Required dependencies can be reproduced from repository configuration.

### Documentation

* OpenAPI is updated when the API changes.
* SRS is updated when requirements change.
* HLD is updated when architecture changes.
* ADR is created for significant architectural decisions.
* Deployment/security documentation is updated when relevant.

### Git Workflow

The normal development workflow has been followed:

```text
branch
↓
implementation
↓
commit
↓
push
↓
Pull Request
↓
review
↓
fix findings
↓
merge into dev
```

### Understanding

The developer shall be able to explain:

* what was implemented
* why it was implemented
* how it works
* how it was tested
* what security considerations apply

A feature that works but cannot be explained is not considered fully Done for this learning project.

---

## 7.2 Project Phase Definition of Done

A project phase is considered Done when:

* its required features are completed
* mandatory tests pass
* CI is green
* relevant documentation is current
* the environment can be reproduced
* the result can be demonstrated
* major known defects are documented or resolved
* the developer understands the introduced technologies

For infrastructure phases, it must additionally be possible to verify that the infrastructure actually performs its intended function.

Examples:

```text
Monitoring
→ metrics can actually be observed

Logging
→ application logs can actually be searched

Alerting
→ an alert can actually be triggered

Backup
→ data can actually be restored

Authentication
→ unauthorized access is actually rejected

Network isolation
→ protected services are actually unreachable externally
```

---

## 7.3 Version 1 Definition of Done

Version 1 is considered complete when:

1. All `Must` requirements are implemented.

2. A user can authenticate successfully.

3. A user can create a Project.

4. The Project creator becomes Project Owner.

5. The Project Owner can add and remove Members.

6. Unauthorized users cannot access the Project.

7. Project Members can create User Stories.

8. Tasks can be created inside User Stories.

9. Tasks can be assigned to Project Members.

10. Tasks can be completed.

11. User Stories can be selected for a Sprint.

12. One Sprint can be used as the Current Sprint.

13. Current Sprint User Stories appear on the Kanban Board.

14. User Stories can be moved between Kanban Columns.

15. Persistent data survives application restart.

16. Database migrations can recreate the required schema.

17. The REST API is documented using OpenAPI.

18. Backend and Frontend builds succeed automatically.

19. Required automated tests pass.

20. Security-sensitive functionality is tested.

21. Secrets are not stored in the repository.

22. The application can be started using the documented development procedure.

23. The system can be deployed to the project's Linux server environment.

24. The documentation reflects the implemented system.

25. Another developer should be able to understand how to build, start and use the application from the repository and documentation.

The project is not considered complete merely because the application starts.

It must be:

```text
Functional
+
Tested
+
Secure enough for its intended environment
+
Reproducible
+
Documented
+
Explainable
```
