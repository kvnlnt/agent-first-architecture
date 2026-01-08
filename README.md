# Architecture

### An AI‑Legible Codebase Architecture

**Purpose**: This document defines a directory convention and architectural philosophy optimized for **AI‑assisted software maintenance**, not merely human comprehension. Traditional architectures (MVC, DDD, Clean Architecture) optimize for human teams navigating complexity. This architecture optimizes for **machine legibility, local reasoning, deterministic change, and safe automation**, while keeping humans in authority over intent and policy.

### Composition Model

The fundamental actors and units in an AI‑first architecture are:

1. Supervisor: The fundamental actor is the human supervisor who the prime-mover who provides high-level intent and oversight
1. Planning Agent: The fundamental entry point is the planning agent that translates intent into actionable tasks
1. Task Agents: The fundamental change agent is highly specialized task agents that can write and modify code
1. Codebase: The fundamental unit of behavior is the codebase that encapsulates all functionality
1. Function: The fundamental unit of behavior is a function
1. Module: The fundamental unit of organization is a module (file)
1. Feature: The fundamental unit of change is a feature with a version
1. Workflow: The fundamental unit of delivery is a workflow (collection of features and orchestrations)
1. Goal: The fundamental unit of achievement is a goal (collection of workflows)

## CRUD first architecture

A CRUD-first architecture is one that establishes a base “postback” CRUD (Create, Read, Update, Delete) interface for all data models before layering on additional business logic or interactive features. This ensures that fundamental data operations are consistent, well-defined, and fully tested across the application, providing a stable foundation for development. Each data model should have a corresponding CRUD interface implemented early in the development process, ideally with 100% test coverage, to facilitate maintainability, scalability, and future integration with other systems.

Once a robust CRUD layer is in place, SPA-like features can be built selectively on top of it. These features leverage the existing CRUD operations, forms, validation, data fetching, and state management to deliver richer, interactive user experiences without duplicating effort or introducing inconsistencies. Because they enhance the experience in a surgical, in-context way rather than replacing the underlying architecture, these lightweight, targeted enhancements can be thought of as “Micro-SPA Augments”—small, reusable interactive modules that bring SPA behavior to specific parts of an otherwise traditional MPA.

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
│   │   ├── contract.yaml
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
```

---

### The Feature Cell (Primary Unit of Change)

Each feature directory is a **closed change cell**.

### Rules

| Constraint   | Requirement                |
| ------------ | -------------------------- |
| Size         | <300 LOC total             |
| Imports      | core/\* only               |
| State        | Explicit inputs only       |
| Side effects | Declared in contract       |
| Tests        | Local, deterministic       |
| Ownership    | Entire directory is atomic |

---

### Example: `features/user.create.v1/`

```
user.create.v1/
├── contract.yaml
├── handler.ts
├── validate.ts
├── tests.ts
└── README.md
```

#### `contract.yaml`

```yaml
name: user.create
version: 1
inputs:
  email: string
  password: string
rules:
  - password.length >= 12
side_effects:
  - create_user_record
  - send_welcome_email
outputs:
  user_id: uuid
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

### Adapters (Edges Only)

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

These create **non‑local reasoning requirements**, which AI handles poorly.

---

### Change Workflow (AI‑First)

1. Contract change proposed
2. AI regenerates handlers, validators, tests
3. CI verifies invariants
4. Humans review **intent**, not mechanics

AI proposes; humans decide.

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

### Planner

This agent is responsible for defining the vision, strategy, and roadmap for a software product. It acts as the liaison between stakeholders and the development team, ensuring that the product meets customer needs and business objectives. Using deep market analytics and research, it prioritizes features, manages the product backlog, and communicates the product goals to the team.

- logs all work to worklogs/product_manager.md
- Defines product vision and strategy
- Prioritizes product features and backlog
- Collaborates with stakeholders to gather requirements
- Communicates product goals to the development team
- Monitors product performance and user feedback

### Business Analyst

This agent is responsible for analyzing business processes, identifying opportunities for improvement, and translating business requirements into technical specifications. It works closely with stakeholders to understand their needs and ensures that the development team has a clear understanding of the project goals.

- logs all work to worklogs/business_analyst.md
- Looks for opportunities to collect feedback within customer facing workflows
- Analyzes business processes and workflows
- Gathers and documents business requirements
- Translates business needs into technical specifications
- Facilitates communication between stakeholders and development team
- Identifies opportunities for process improvement
- Tracks reported engineering issues and coordinates with relevant agents to ensure timely resolution

### Systems Architect

This agent is responsible for designing and overseeing the overall architecture of complex software systems. It ensures that the system components work together seamlessly, meet performance and scalability requirements, and align with business goals.

- logs all work to worklogs/systems_architect.md
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

- logs all work to worklogs/platform_engineer.md
- Designs and implements platform architecture
- Manages cloud infrastructure and services
- Automates deployment and scaling processes
- Monitors platform performance and reliability
- Collaborates with development teams to optimize application performance
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Frontend Engineer

This agent is responsible for developing and maintaining the user interface and user experience of web applications. It focuses on creating responsive, accessible, and visually appealing designs that enhance user engagement.

- logs all work to worklogs/frontend_engineer.md
- Develops responsive web interfaces using modern frameworks
- Ensures cross-browser compatibility and accessibility
- Implements UI/UX designs and collaborates with designers
- Optimizes frontend performance and load times
- Integrates frontend with backend services via APIs
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Backend Engineer

This agent is responsible for developing and maintaining the server-side logic, databases, and APIs of web applications. It ensures that the backend systems are robust, scalable, and secure, supporting the needs of the frontend and overall application functionality.

- logs all work to worklogs/backend_engineer.md
- Develops server-side logic and APIs
- Manages database design and optimization
- Ensures application security and data integrity
- Implements business logic and workflows
- Collaborates with frontend engineers to integrate services
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### Database Administrator

This agent is responsible for the design, implementation, maintenance, and optimization of database systems. It ensures data integrity, security, and availability while supporting the needs of applications and users.

- logs all work to worklogs/database_administrator.md
- Designs and implements database schemas
- Monitors and optimizes database performance
- Ensures data security and integrity
- Manages backups and disaster recovery
- Supports application development with database expertise
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### DevOps Engineer

This agent is responsible for managing the infrastructure, deployment, and operational aspects of software applications. It ensures that systems are reliable, scalable, and secure, while also optimizing performance and cost-efficiency.

- logs all work to worklogs/devops_engineer.md
- Manages cloud infrastructure and services
- Implements CI/CD pipelines
- Monitors system performance and reliability
- Handles incident response and disaster recovery
- Automates operational tasks and processes
- Logs issues and errors to the engineering issue tracker for visibility and resolution

### QA Engineer

This agent is responsible for ensuring the quality and reliability of software applications through systematic testing and validation. It designs and executes test plans, identifies defects, and collaborates with development teams to improve product quality.

- logs all work to worklogs/qa_engineer.md
- Designs and executes test plans
- Identifies and documents defects
- Collaborates with developers to resolve issues
- Automates testing processes
- Ensures compliance with quality standards
- Logs issues and errors to the engineering issue tracker for visibility and resolution
