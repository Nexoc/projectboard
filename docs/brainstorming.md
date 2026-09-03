# Brainstorming. not SRS or HLD
### What is the application?
- It will be a Kanban board with some features

### Why are we building it?
- I will use this tool myself and maybe someone else too.
- For my future team.

### Who will use it?
- Development teams

### What are the main features?
- user can create a project and automatically becomes a Project Owner
- Project Owner can update/delete own project

- Project Owner can add registered users to a project
- Project Owner can remove users from a project

- user can create/update/delete *sprints*
- user can assign users to *tasks*
- user can see the current *sprint*

- user can create/update/delete *columns*

- user can create/update/delete *cards*
- user can move *cards*
- user can add/update *tasks* to a card

- user can *log in/register*

- user can create/update/delete *user stories*

- user can create a sprint *retrospective*
- user can add positive/negative *feedback*
- user can add improvement actions for the next sprint

- user can select interface language *English / German*

### What is NOT included in the first version?
- AI
- realtime updates
- file attachments
- chat


### Project
- Backlog
  - User Stories
    - Tasks
- Sprint
  - selected Tasks
  - start date + end date
  - retrospective
- Kanban Board
(create/update/delete columns)
(assign users)
(add/update tasks to a card)
  - ToDo
  - In Progress
  - Done (send notification)


Global:
- ADMIN
  - sees registered users
  - manages global user roles
- USER
  - Project A: Project Owner
  - Project B: MEMBER

Project:
- Project Owner
  - sees registered users
  - adds/removes selected users to/from the project
- MEMBER


### Future AI Feature

* Local LLM can analyze a User Story
* AI can suggest tickets/tasks based on the User Story
* AI can suggest acceptance criteria
* generated tickets are not created automatically
* user must review and confirm the suggestions first

Flow:

User Story
→ Local LLM
→ Suggested Tickets / Tasks
→ User Review
→ Accept
→ Add to Backlog
