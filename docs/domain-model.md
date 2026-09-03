# ProjectBoard Domain Model
| Field | Value |
| --- | --- |
| Status | Draft |
| Parent | [High-Level Design](hld.md) |
| Requirements | [SRS](srs.md) |
| ERD | [Conceptual ERD](diagrams/erd.md) |
| Last updated | 2026-09-03 |

This document defines Version 1 domain concepts, ownership, roles, and invariants.
Physical persistence, API contracts, and messaging topology belong to dedicated designs.

# 1. Domain Overview
`Project` is the main business, authorization, and data-isolation boundary.

```text
Project
├── ProjectMembers
├── UserStories
│   └── Tasks
├── Sprints
│   └── selected UserStories
└── BoardColumns
    └── positioned UserStories
```

Version 1 rules:
- Backlog is a logical Project view, not a persistent entity.
- UserStory is the Kanban card; no separate `Card` entity exists.
- Task belongs to one UserStory.
- Sprint selects UserStories, not Tasks.
- BoardColumns organize UserStories.
- Task assignee and `Current Sprint` are outside Version 1.

# 2. Core Concepts
| Concept | Meaning |
| --- | --- |
| Identity Reference | Local reference to a Keycloak identity |
| Project | Workspace and authorization boundary |
| ProjectMember | Identity-to-Project membership with role |
| UserStory | Planning item and Kanban card |
| Task | Work item inside one UserStory |
| Sprint | Planning interval selecting UserStories |
| BoardColumn | Workflow stage for UserStories |
| Activity/Audit | Historical business activity |

Keycloak owns authentication credentials.
ProjectBoard owns Project membership and business authorization.

# 3. Identity and Membership
```text
Keycloak Identity → Identity Reference → ProjectMember → Project
```

ProjectBoard stores no passwords or Keycloak credentials.

Version 1 roles:
```text
PROJECT_OWNER
MEMBER
```

Any authenticated identity may create a Project; the creator becomes `PROJECT_OWNER`.
A user may have different roles in different Projects.
There is no global ProjectBoard `ADMIN`.
Ownership transfer and advanced permissions are outside Version 1.

# 4. Project
A Project owns memberships, UserStories, Sprints, BoardColumns, and related Activity/Audit context.
Project creation creates the initial owner membership.
Project resources must never cross Project authorization boundaries.
Version 1 uses hard deletion; physical deletion rules belong to `database-design.md`.

# 5. UserStory and Backlog
Every UserStory belongs to exactly one Project.
A UserStory:
- cannot move between Projects;
- is the primary planning item and Kanban card;
- may contain Tasks;
- may participate in Sprint planning;
- may be positioned in a BoardColumn.

Backlog is a logical Project view over UserStories.
No duplicate Backlog copy of a UserStory exists.
Exact Backlog filtering remains open.

# 6. Task
Every Task belongs to exactly one UserStory and inherits its Project context from that UserStory.
Tasks are not independently selected into Sprints.
Task assignment to Project members is outside Version 1.

# 7. Sprint
Every Sprint belongs to exactly one Project.
A Sprint may select only UserStories from the same Project.
Version 1 has no dedicated `Current Sprint`.
Sprint dates do not automatically start or complete a Sprint.
Advanced Sprint analytics are outside Version 1.

# 8. Kanban Board
The Board is a Project workflow view organized by persistent BoardColumns.

New Projects initially contain:
```text
To Do
In Progress
Done
```

Project members may create columns, rename columns, delete empty columns, and move UserStories.
Column order and UserStory placement are persistent.
A UserStory may only use a BoardColumn from the same Project.
No persistent `Board` entity is required.

# 9. Module Ownership
The Backend is a Modular Monolith.

| Module | Responsibility |
| --- | --- |
| `project` | Project lifecycle |
| `membership` | Memberships and roles |
| `story` | UserStory lifecycle |
| `task` | Task lifecycle |
| `sprint` | Sprint lifecycle and Story selection |
| `board` | Columns, placement, ordering |
| `shared` | Technical primitives only |

Rules:
1. each module owns its rules and persistence;
2. modules communicate through explicit application interfaces;
3. modules do not manipulate another module's repository directly;
4. cross-module use cases are coordinated in the application layer;
5. one PostgreSQL transaction may span modules when immediate consistency is required;
6. `shared` owns no business domain.

# 10. Authorization
| Capability | PROJECT_OWNER | MEMBER | Non-member |
| --- | :---: | :---: | :---: |
| View Project | Yes | Yes | No |
| Update/Delete Project | Yes | No | No |
| Manage membership | Yes | No | No |
| Work with Stories/Tasks/Sprints | Yes | Yes | No |
| Manage Board workflow | Yes | Yes | No |

All protected operations are authorized by the Backend.
Frontend visibility is never an authorization mechanism.
A resource identifier alone never grants access.

# 11. Core Invariants
1. Project is the authorization and isolation boundary.
2. Project creation creates the creator's `PROJECT_OWNER` membership.
3. One identity has at most one membership per Project.
4. Membership role is `PROJECT_OWNER` or `MEMBER`.
5. UserStory belongs to exactly one Project and cannot move between Projects.
6. UserStory is the Kanban card; no `Card` entity exists.
7. UserStory uses optimistic locking.
8. Task belongs to exactly one UserStory and inherits its Project.
9. Tasks are never independently selected into Sprints.
10. Sprint belongs to one Project and selects only same-Project UserStories.
11. BoardColumn belongs to one Project.
12. New Projects contain `To Do`, `In Progress`, and `Done`.
13. Non-empty BoardColumns cannot be deleted.
14. BoardColumn order and UserStory placement are persistent.
15. UserStory and target BoardColumn must belong to the same Project.
16. Protected operations resolve the authoritative owning Project.
17. Business authorization is enforced by the Backend.

# 12. Concurrency
UserStory editing uses:
```text
Redis Edit Lock + PostgreSQL Optimistic Lock
```

Redis provides temporary collaboration coordination with TTL and renewal.
It does not grant authorization or guarantee data integrity.
PostgreSQL optimistic locking rejects stale writes and is authoritative.

# 13. Business Events and Audit
Completed operations may emit events such as:
```text
ProjectCreated
MemberAdded
UserStoryCreated
TaskCompleted
```

Events may asynchronously materialize Activity/Audit history.
Business events do not replace synchronous domain validation.
Reliable delivery belongs to `messaging-design.md`.

Activity/Audit is append-only through normal APIs and remains separate from technical and security logs.

# 14. Version 1 Scope
Core domain concepts:
```text
Identity Reference
Project
ProjectMember
UserStory
Task
Sprint
BoardColumn
Activity/Audit
```

Supporting technical concepts include Outbox Event, Edit Lock, and Correlation ID.

Outside Version 1:
- global ADMIN;
- Card entity;
- Task assignee;
- Current Sprint;
- Retrospective;
- notifications;
- ownership transfer;
- advanced permissions;
- advanced Sprint analytics;
- soft delete;
- active AI functionality.

# 15. Open Decisions
1. Task as separate aggregate root versus inside UserStory aggregate.
2. Physical Sprint-to-UserStory association.
3. Exact Backlog filtering.
4. Physical Board placement and ordering representation.
5. Optimistic locking beyond UserStory.
6. Hard-delete cascade/restrict behavior.
7. Relationship between Sprint filtering and Board presentation.
8. Identity Reference lifecycle after Keycloak identity disable/delete.