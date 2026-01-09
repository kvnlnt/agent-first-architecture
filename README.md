# Architecture

### An AI‑Legible Codebase Architecture

**Purpose**: This document defines a directory convention and architectural philosophy optimized for **AI‑assisted software maintenance**, not merely human comprehension. Traditional architectures (MVC, DDD, Clean Architecture) optimize for human teams navigating complexity. This architecture optimizes for **machine legibility, local reasoning, deterministic change, and safe automation**, while keeping humans in authority over intent and policy.

### Composition Model

The fundamental actors and units in an AI‑first architecture are:

1. Supervisor: The fundamental actor in the system is a human supervisor who is the prime-mover who provides high-level intent and oversight
2. Planner Agent: The planner agent translates Supervisor intent into actionable tasks
3. Orchestrator Agent: The orchestrator agent coordinates the execution of tasks by task agents
4. Task Agents: highly specialized task agents that use task documents written by the Planner Agent to write and modify code
5. Codebase: The fundamental structure of the system is the codebase itself, organized to facilitate AI understanding and modification
6. Function: The fundamental unit of behavior. They should be small, pure, and side-effect free. Arguments should always be an explicit input object so that future inputs can be added without changing the signature.
7. Module: The fundamental unit of organization is a module (file). Functions are grouped into modules based on cohesive functionality. High cohesion, low coupling.
8. Feature: The fundamental unit of change is a feature with a version
9. Workflow: The fundamental unit of delivery is a workflow (collection of features and orchestrations)
10. Goal: The fundamental unit of achievement is a goal (collection of workflows)

### Core Philosophy

### 1. Authority Is Declarative, Not Emergent

- **Specifications are the source of truth**
- Code is a _derivative artifact_
- Tests assert contracts, not implementations

AI operates best when intent is explicit and machine‑readable.

---

### 2. Optimize for Local Reasoning

A change should require **no more than ~3 files** to fully understand.

If modifying behavior requires tracing through shared utilities, abstract base classes, or implicit framework magic, the system is hostile to AI.

---

### 3. Duplication Is Cheaper Than Ambiguity

- Storage is cheap
- Human cleverness is expensive
- AI reasoning errors are catastrophic

**Intentional duplication** prevents cascade failures and allows AI to regenerate behavior safely from specs.

---

### 4. Explicit Boundaries Beat Abstractions

- No “magic” dependency injection
- No reflection‑heavy frameworks
- No cross‑feature imports

Boundaries are enforced by **directory structure and tooling**, not conventions or discipline.

---

### Directory Convention

```
/
├── adapters/           # IO boundaries only
│   ├── http/
│   ├── cli/
│   └── queue/
|
├── core/               # Human‑owned, AI‑read‑only
|   ├── agents/         # Agent definitions
|   ├── config/         # Global configuration
│   ├── policy/         # Hard rules, invariants, constraints
│   ├── types/          # Global primitive types
│   └── runtime/        # Bootstrapping, wiring, lifecycle
|
├── docs/               # Human context & decisions
│
├── features/           # AI‑modifiable change cells
│   ├── user.create.v1/
│   │   ├── contract.ts
│   │   ├── handler.ts
│   │   ├── validate.ts
│   │   ├── tests.ts
│   │   └── README.md
│   │
│   ├── user.create.v2/
│   │   └── ...
│   │
│   └── order.cancel.v1/
│       └── ...
│
├── generated/          # Machine‑generated artifacts
│   └── (never edited by humans)
│
├── scripts/            # Maintenance + orchestration
|
└── tasks/              # Tasks genereated by planning agent and executed by task agents
    └── todo/           # To do tasks
    └── issues/         # Reported issues
    └── done/           # Completed tasks
```

---

### The Feature Cell (Primary Unit of Change)

Each feature directory is a **closed change cell**.

### Rules

| Constraint   | Requirement                |
| ------------ | -------------------------- |
| Size         | <1000 LOC total            |
| Imports      | core/\* only               |
| State        | Explicit inputs only       |
| Side effects | Declared in contract       |
| Tests        | Local, deterministic       |
| Ownership    | Entire directory is atomic |

---

### Example: `features/user.create.v1/`

```
user.create.v1/
├── contract.ts
├── handler.ts
├── validate.ts
├── tests.ts
└── README.md
```

#### `contract.ts`

```typescript
export const contract: Contract = {
  name: "user.create",
  version: 1,
  inputs: {
    email: "string",
    password: "string",
  },
  rules: [{ rule: "password.length >= 12" }],
  side_effects: ["create_user_record", "send_welcome_email"],
  outputs: {
    user_id: "uuid",
  },
};
```

> This file is **authoritative**.

---

#### `handler.ts`

- Implements exactly what the contract specifies
- No hidden behavior
- No branching on environment or flags

---

#### `validate.ts`

- Local validation only
- No shared validators
- Regenerated when contract changes

---

#### `tests.ts`

- Asserts contract compliance, not implementation details
- Tests inputs/outputs match contract declarations
- Validates all declared rules are enforced
- Confirms side effects are triggered (via stubs/mocks)
- Local and deterministic—no external dependencies
- Regenerated when contract changes

---

### Versioned Behavior (No Conditionals)

Instead of:

```ts
if (featureFlag) {
  // new behavior
}
```

Use:

```
user.create.v1/
user.create.v2/
```

Benefits:

- AI can diff versions mechanically
- Safe backports and forward‑ports
- Deletion is explicit and auditable

---

### When to Version

Create a new version (`v2`, `v3`, etc.) when:

- Changing inputs/outputs in `contract.ts`
- Modifying declared rules
- Adding/removing side effects
- Altering the fundamental behavior of the feature

Do NOT version for:

- Bug fixes that don't change the contract
- Performance optimizations
- Internal refactoring within the handler
- Code style or formatting changes

---

### Core vs Features

### `/core`

- **Never modified by AI**
- Contains invariants, policies, and lifecycles
- Small, boring, stable

Examples:

- Authorization rules
- Deployment constraints
- Domain primitives

---

### `/features`

- Fully AI‑modifiable
- All business logic lives here
- Designed for regeneration

AI failures are _contained_.

---

### `/adapters` (Edges Only)

Adapters translate the outside world into feature calls.

Rules:

- No business logic
- No branching beyond IO concerns
- Thin, dumb, disposable

Examples:

- HTTP request → feature input
- CLI args → feature input
- Queue message → feature input

---

### Naming Conventions (Mechanical Symmetry)

Bad (human‑clever):

```
engine.ts
resolver.ts
orchestrator.ts
```

Good (AI‑legible):

```
user.create.handler.ts
user.create.validate.ts
user.create.tests.ts
```

Benefits:

- Pattern completion
- Safe glob edits
- Deterministic discovery

---

### What Is Explicitly Forbidden

- `utils/`
- Shared helpers across features
- Implicit globals
- Framework magic
- Reflection
- Runtime behavior not declared in contracts
- Cross‑feature imports
- Circular dependencies
- Hidden state
- Aspect‑oriented programming
- Monolithic functions (>50 LOC)
- High‑arity functions (>3 args)
- Large feature cells (>300 LOC)
- Complex conditionals based on environment, flags, or context
- Runtime configuration changes that alter behavior

These create **non‑local reasoning requirements**, which AI handles poorly.

---

### Fullstack Implementation Layers

The AI-first architecture establishes two distinct implementation layers for fullstack web applications. This layered approach ensures that fundamental operations are stable, well-tested, and AI-manageable before adding complexity.

---

### Foundation Layer (CRUD-First)

The Foundation Layer establishes a complete, testable interface for all data models before any interactive features are added. This layer is **fully generatable** from model specifications, making it ideal for AI implementation.

#### Backend

Each model generates a standard REST API:

| Method | Endpoint           | Purpose                                            |
| ------ | ------------------ | -------------------------------------------------- |
| POST   | `/api/{model}`     | Create a new record                                |
| GET    | `/api/{model}`     | List records (with pagination, filtering, sorting) |
| GET    | `/api/{model}/:id` | Retrieve a specific record                         |
| PUT    | `/api/{model}/:id` | Replace a record                                   |
| PATCH  | `/api/{model}/:id` | Partially update a record                          |
| DELETE | `/api/{model}/:id` | Delete a record                                    |

Standard capabilities:

- Authentication and authorization
- Input validation and business rules
- Audit logging
- Error handling with appropriate HTTP status codes
- Database interactions

#### Frontend

Each model generates resourceful routes:

| Route                 | Purpose                                  |
| --------------------- | ---------------------------------------- |
| `/{model}`            | List view with search and create options |
| `/{model}/new`        | Create form                              |
| `/{model}/:id`        | Detail view with edit/delete options     |
| `/{model}/:id/edit`   | Edit form                                |
| `/{model}/:id/delete` | Delete confirmation                      |

Standard capabilities:

- Form components with validation
- List components with pagination
- State management
- API integration

#### Why Foundation First?

- **100% test coverage** before adding complexity
- **Consistent patterns** across all models
- **AI-generatable** from model specifications
- **Stable base** for enhancement features
- **Rollback safety**—enhancements can be removed without breaking core functionality

---

### Enhancement Layer (Progressive)

The Enhancement Layer builds **on top of** the Foundation Layer, adding richer interactions without replacing or duplicating core CRUD operations. These are surgical, in-context improvements—not a separate architecture.

#### Backend Enhancements

- Bulk operations (e.g., batch create/update/delete)
- Complex queries (aggregations, joins, full-text search)
- Rate limiting and advanced security
- Caching strategies
- Integration with analytics, notifications, third-party services

#### Frontend Enhancements

**Quick Access**

- Keyboard shortcuts
- Context menus
- Command palettes

**Inline Interactions**

- Quick edit (inline editing, modals)
- Drag-and-drop reordering
- Real-time updates (WebSocket, SSE)

**Cross-Model Features**

- Linked/nested resources
- Dashboards spanning multiple models
- Bulk actions across models
- Cross-model validation
- Unified reporting

**Context-Aware Behavior**

- Role-based UI adaptation
- Event-driven notifications
- Multi-step workflow guidance

**Advanced Components**

- Data visualizations
- Rich editors
- Third-party integrations

#### Enhancement Rules

| Constraint  | Requirement                                                  |
| ----------- | ------------------------------------------------------------ |
| Dependency  | Must use Foundation Layer APIs, not bypass them              |
| Isolation   | Enhancement failures must not break Foundation functionality |
| Testability | Enhancements have separate test suites                       |
| Degradation | System remains functional if enhancements are disabled       |

---

### Layer Relationship

```
┌─────────────────────────────────────────────────────┐
│           Enhancement Layer (Progressive)           │
│  Dashboards, Bulk Ops, Real-time, Cross-model UX   │
├─────────────────────────────────────────────────────┤
│            Foundation Layer (CRUD-First)            │
│   REST APIs, Resourceful Routes, Forms, Lists      │
├─────────────────────────────────────────────────────┤
│                   Data Models                        │
│            Schemas, Contracts, Types                │
└─────────────────────────────────────────────────────┘
```

**Key insight**: The Foundation Layer is the **source of truth** for data operations. The Enhancement Layer consumes it—never circumvents it.

---

---

### Change Workflow (AI‑First)

1. Human Supervisor makes a request to the Planner Agent
2. Planner Agent breaks down the request into discrete tasks and stores each in the `/tasks/todo/` directory with the following naming convention: `[priority]_[task_agent]_[short_description].md` (e.g., `001_backend-engineer_implement-user-create-handler.md`). Task dependencies can be specified in YAML frontmatter within each task file. All features should be implemented in degrees of complexity, starting with a minimal viable version and iterating towards the full specification. The planner agent should ensure that each task is small enough to be completed by a single task agent in one iteration and report back if the task is too large or complex.
3. Human Supervisor reviews the tasks in the `/tasks/todo/` directory
4. Human Supervisor requests the Orchestrator Agent to execute the tasks in the `/tasks/todo/` directory in order and to report back on completion and any issues encountered. The Orchestrator Agent carries out each task via a task agent in order of priority. It halts execution if any task is moved to the `/tasks/issues/` directory for Human Supervisor review.
5. Each Task Agent picks up its assigned task from the `/tasks/todo/` directory. All work performed by Task Agents is appended to the original task document in `/tasks/todo/` and moved to `/tasks/done/` upon completion. If issues are encountered, all work performed, issues, and proposed solutions are documented in the original task document, which is then moved to `/tasks/issues/` for Human Supervisor review.
6. Upon completion of all tasks, the Orchestrator Agent generates a summary report of the changes made, tests executed, and any issues encountered during the process. This report is stored as `/docs/reports/[YYYY-MM-DD]_[workflow-name].md` for future reference and auditing.
7. Continuous Integration (CI) system runs automated tests and verifies that all contracts and invariants are upheld. If any tests fail or invariants are violated, the CI system generates a report and notifies the Human Supervisor for review.
8. Human Supervisor reviews the changes, test results, and CI reports to ensure that the modifications align with the original intent and business goals. If everything is satisfactory, the changes are approved for deployment.
9. If any issues were identified during the review, the Human Supervisor may request further modifications or clarifications, which would initiate a new cycle of task creation and execution as needed.
10. Once approved, the changes are deployed to the production environment following established deployment procedures.
11. Post-deployment, the system is monitored to ensure stability and performance, with any anomalies reported back to the Human Supervisor for further action.
12. All task documents, reports, and related artifacts are archived in the `/docs/` directory for future reference and auditing purposes.

AI proposes; humans decide.

---

### Regression Prevention

The architecture ensures regressions are detected and prevented through:

- **Immutable Versions**: Each feature version is immutable once deployed; behavior changes require new versions
- **Contract Diffing**: CI compares `contract.ts` across versions to detect breaking changes
- **Cross-Version Testing**: Tests from previous versions can be run against new versions to verify backward compatibility when required
- **Local Test Isolation**: Each feature's tests are self-contained and deterministic, ensuring failures are traceable to specific features
- **Invariant Enforcement**: CI validates that all contracts and policy invariants are upheld before deployment

---

### Mental Model Shift

**Old**:

> Architecture helps humans understand systems

**New**:

> Architecture helps machines safely change systems

Humans steward meaning.
Machines steward consistency.

---

### Design North Star

> _If an AI cannot safely modify one feature without understanding the entire codebase, the architecture has already failed._

---

### Appendix: When This Architecture Is a Bad Fit

- Highly exploratory research code
- Performance‑critical inner loops
- Novel algorithm design

In those cases, keep AI advisory, not authoritative.

---

## Agents

### Supervisor

This agent is the human overseer and prime-mover who provides high-level intent, reviews AI-generated changes, and ensures alignment with business goals and policies. It acts as the final authority on decisions and maintains accountability for the overall system.

### Planner Agent

This agent is responsible for translating Supervisor intent into actionable tasks. It defines the vision, strategy, and roadmap for a software product, acting as the liaison between stakeholders and the development team. Using deep market analytics and research, it prioritizes features, manages the product backlog, and communicates the product goals to the team.

- Defines product vision and strategy
- Prioritizes product features and backlog
- Collaborates with stakeholders to gather requirements
- Communicates product goals to the development team
- Monitors product performance and user feedback

### Orchestrator

This agent is responsible for coordinating and managing the execution of tasks by various specialized task agents. It ensures that tasks are completed in the correct order, dependencies are managed, and overall project timelines are adhered to. The Orchestrator acts as the central hub for task execution, monitoring progress, and handling any issues that arise during the process.

- Coordinates task execution among specialized agents
- Manages task dependencies and timelines
- Monitors progress and reports status to the Supervisor
- Handles issues and escalates when necessary
- Ensures overall project coherence and alignment with goals

### Business Analyst

This agent is responsible for analyzing business processes, identifying opportunities for improvement, and translating business requirements into technical specifications. It works closely with stakeholders to understand their needs and ensures that the development team has a clear understanding of the project goals.

- Analyzes business processes and workflows
- Gathers and documents business requirements
- Translates business needs into technical specifications
- Facilitates communication between stakeholders and development team
- Identifies opportunities for process improvement
- Tracks reported engineering issues and coordinates with relevant agents to ensure timely resolution

### Systems Architect

This agent is responsible for designing and overseeing the overall architecture of complex software systems. It ensures that the system components work together seamlessly, meet performance and scalability requirements, and align with business goals.

- Is a proponent of Hex Architecture and Domain-Driven Design
- Is a proponent of Monorepo and CI/CD best practices
- Is a proponent of 12 Factor App methodology
- Designs system architecture and component interactions
- Ensures scalability, performance, and reliability
- Bias towards rapid feature delivery while ensuring long-term sustainability
- Define the overall system architecture and technical strategy
- Select and justify the technology stack (languages, frameworks, databases, tooling)
- Maintain a holistic view of system components and their interactions
- Decide hosting and infrastructure models (cloud, bare metal, hybrid, cost tradeoffs)
- Design DevOps workflows (CI/CD, environments, secrets, rollbacks, observability)
- Establish QA and testing strategy (unit, integration, e2e, load, regression)
- Define standards for reliability, scalability, security, and maintainability
- Ensure all components interoperate cleanly and predictably
- Balance speed of delivery against long-term technical debt
- Document architectural decisions and their rationale
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Platform Engineer

This agent is responsible for building and maintaining the underlying platform that supports software applications. It focuses on creating a stable, scalable, and efficient environment for application deployment and operation.

- Designs and implements platform architecture
- Manages cloud infrastructure and services
- Automates deployment and scaling processes
- Monitors platform performance and reliability
- Collaborates with development teams to optimize application performance
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Frontend Engineer

This agent is responsible for developing and maintaining the user interface and user experience of web applications. It focuses on creating responsive, accessible, and visually appealing designs that enhance user engagement.

- Develops responsive web interfaces using modern frameworks
- Ensures cross-browser compatibility and accessibility
- Implements UI/UX designs and collaborates with designers
- Optimizes frontend performance and load times
- Integrates frontend with backend services via APIs
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Backend Engineer

This agent is responsible for developing and maintaining the server-side logic, databases, and APIs of web applications. It ensures that the backend systems are robust, scalable, and secure, supporting the needs of the frontend and overall application functionality.

- Develops server-side logic and APIs
- Manages database design and optimization
- Ensures application security and data integrity
- Implements business logic and workflows
- Collaborates with frontend engineers to integrate services
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Database Administrator

This agent is responsible for the design, implementation, maintenance, and optimization of database systems. It ensures data integrity, security, and availability while supporting the needs of applications and users.

- Designs and implements database schemas
- Monitors and optimizes database performance
- Ensures data security and integrity
- Manages backups and disaster recovery
- Supports application development with database expertise
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### DevOps Engineer

This agent is responsible for managing the infrastructure, deployment, and operational aspects of software applications. It ensures that systems are reliable, scalable, and secure, while also optimizing performance and cost-efficiency.

- Manages cloud infrastructure and services
- Implements CI/CD pipelines
- Monitors system performance and reliability
- Handles incident response and disaster recovery
- Automates operational tasks and processes
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### QA Engineer

This agent is responsible for ensuring the quality and reliability of software applications through systematic testing and validation. It designs and executes test plans, identifies defects, and collaborates with development teams to improve product quality.

- Designs and executes test plans
- Identifies and documents defects
- Collaborates with developers to resolve issues
- Automates testing processes
- Ensures compliance with quality standards
- Logs issues and errors to the engineering issue tracker for visibility and resolution
