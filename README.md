# Architecture

## An AI‑Legible Codebase Architecture

**Purpose**: This document defines a directory convention and architectural philosophy optimized for **AI‑assisted software maintenance**, not merely human comprehension. Traditional architectures (MVC, DDD, Clean Architecture) optimize for human teams navigating complexity. This architecture optimizes for **machine legibility, local reasoning, deterministic change, and safe automation**, while keeping humans in authority over intent and policy.

## Composition Model

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

---

## Core Philosophy

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

## Directory Convention

```
/
├── adapters/              # IO boundaries only
│   ├── http/
│   │   └── routes.ts      # Maps HTTP routes → feature handlers
│   ├── cli/
│   └── queue/
│
├── core/                  # Human‑owned, AI‑read‑only
│   ├── agents/            # Agent definitions
│   ├── config/            # Global configuration
│   ├── db/                # Database access layer
│   │   ├── client.ts      # Database client/connection
│   │   ├── migrations/    # Schema migrations
│   │   └── repositories/  # Data access objects per model
│   ├── lib/               # Tested, contract-defined primitives
│   │   ├── validators/    # Email, UUID, phone, etc.
│   │   ├── formatters/    # Date, currency, string transforms
│   │   ├── parsers/       # Input parsing utilities
│   │   └── errors/        # Error types and factories
│   ├── policy/            # Hard rules, invariants, constraints
│   ├── schema/            # Canonical domain model (single source of truth)
│   │   └── canonical.cue  # All entities, constraints, relationships
│   ├── types/             # Global primitive types
│   └── runtime/           # Bootstrapping, wiring, lifecycle
│
├── docs/                  # Human context & decisions
│
├── features/              # AI‑modifiable change cells (entity-first)
│   │
│   ├── user.v1/           # User entity (all operations)
│   │   ├── contract.ts    # Entity contract + operations
│   │   ├── api/
│   │   │   ├── create.handler.ts
│   │   │   ├── read.handler.ts
│   │   │   ├── update.handler.ts
│   │   │   ├── delete.handler.ts
│   │   │   └── list.handler.ts
│   │   ├── page/
│   │   │   ├── list.handler.ts
│   │   │   ├── detail.handler.ts
│   │   │   └── edit.handler.ts
│   │   ├── fragment/
│   │   │   ├── row.handler.ts
│   │   │   └── form.handler.ts
│   │   ├── tests.ts
│   │   └── README.md
│   │
│   ├── order.v1/          # Order entity (with state machine)
│   │   ├── contract.ts    # Entity contract + machine + transitions
│   │   ├── api/
│   │   │   ├── create.handler.ts
│   │   │   └── read.handler.ts
│   │   ├── page/
│   │   │   ├── checkout.handler.ts
│   │   │   └── confirmation.handler.ts
│   │   ├── fragment/
│   │   │   ├── cart-summary.handler.ts
│   │   │   └── line-item.handler.ts
│   │   ├── transition/
│   │   │   ├── submit.handler.ts
│   │   │   ├── confirm.handler.ts
│   │   │   └── cancel.handler.ts
│   │   ├── tests.ts
│   │   └── README.md
│   │
│   └── cart.v1/           # Cart entity (with state machine)
│       ├── contract.ts
│       ├── fragment/
│       ├── transition/
│       │   ├── add-item.handler.ts
│       │   ├── remove-item.handler.ts
│       │   └── review.handler.ts
│       └── tests.ts
│
├── generated/             # Machine‑generated artifacts
│   └── (never edited by humans)
│
├── scripts/               # Maintenance + orchestration
│
└── tasks/                 # Tasks generated by Planner Agent
    ├── todo/              # Queued tasks (unique priorities)
    ├── issues/            # Blocked tasks for Supervisor review
    └── done/              # Completed tasks (archived)
```

---

## The Feature Cell (Primary Unit of Change)

Each feature directory represents a **single entity** and is a **closed change cell** containing all operations for that entity.

### Entity-First Organization

Features are organized by **entity** (the domain object), not by artifact type. Each entity folder contains:

| Subfolder     | Purpose                                                   |
| ------------- | --------------------------------------------------------- |
| `contract.ts` | Entity contract: operations, state machine, transitions   |
| `api/`        | JSON endpoints for programmatic access                    |
| `page/`       | Full HTML document handlers                               |
| `fragment/`   | HTML partial handlers for AJAX                            |
| `transition/` | State machine transition handlers (if entity has machine) |
| `tests.ts`    | All tests for this entity                                 |

### Hypermedia Architecture

This architecture follows **server-side rendering with hypermedia controls**:

- **No client-side state** — Application state lives on the server
- **HTML over the wire** — Server renders all content, client swaps regions
- **State passed per-request** — Client sends current state, server validates and transitions
- **Progressive enhancement** — Pages work without JavaScript; fragments enhance UX

### Rules

| Constraint   | Requirement                                        |
| ------------ | -------------------------------------------------- |
| Size         | <1000 LOC total per entity                         |
| Imports      | `core/*` only (`lib/`, `db/`, `types/`, `policy/`) |
| State        | Explicit inputs only                               |
| Side effects | Declared in contract                               |
| Tests        | Local, deterministic                               |
| Ownership    | Entire directory is atomic                         |

---

### Example: `features/user.v1/` (Entity without State Machine)

```
user.v1/
├── contract.ts
├── api/
│   ├── create.handler.ts
│   ├── read.handler.ts
│   ├── update.handler.ts
│   ├── delete.handler.ts
│   └── list.handler.ts
├── page/
│   ├── list.handler.ts
│   ├── detail.handler.ts
│   └── edit.handler.ts
├── fragment/
│   ├── row.handler.ts
│   └── form.handler.ts
├── tests.ts
└── README.md
```

#### `contract.ts` (Entity without State Machine)

```typescript
// features/user.v1/contract.ts
import type { User, UserCreateInput, UserUpdateInput } from "../../generated/types";

export const contract = {
  name: "user",
  version: 1,

  // No state machine for simple CRUD entities
  machine: null,

  operations: {
    "api.create": {
      input: "UserCreateInput",
      output: "User",
      side_effects: ["create_user_record", "send_welcome_email"],
    },
    "api.read": {
      input: "{ id: UUID }",
      output: "User",
    },
    "api.update": {
      input: "UserUpdateInput",
      output: "User",
      side_effects: ["update_user_record"],
    },
    "api.delete": {
      input: "{ id: UUID }",
      output: "void",
      side_effects: ["delete_user_record"],
    },
    "api.list": {
      input: "{ page?: number, limit?: number }",
      output: "User[]",
    },
    "page.list": {
      input: "{ page?: number }",
      output: "HTML",
    },
    "page.detail": {
      input: "{ id: UUID }",
      output: "HTML",
    },
    "page.edit": {
      input: "{ id: UUID }",
      output: "HTML",
    },
    "fragment.row": {
      input: "{ id: UUID }",
      output: "HTML",
    },
    "fragment.form": {
      input: "{ id?: UUID }",
      output: "HTML",
    },
  },
} as const;
```

---

### Example: `features/order.v1/` (Entity with State Machine)

```
order.v1/
├── contract.ts
├── api/
│   ├── create.handler.ts
│   └── read.handler.ts
├── page/
│   ├── checkout.handler.ts
│   └── confirmation.handler.ts
├── fragment/
│   ├── cart-summary.handler.ts
│   └── line-item.handler.ts
├── transition/
│   ├── submit.handler.ts
│   ├── confirm.handler.ts
│   └── cancel.handler.ts
├── tests.ts
└── README.md
```

#### `contract.ts` (Entity with State Machine)

```typescript
// features/order.v1/contract.ts
import type { Order, OrderItem } from "../../generated/types";
import type { MachineState } from "../../generated/machines";

export const contract = {
  name: "order",
  version: 1,

  // State machine definition lives in the entity contract
  machine: {
    name: "OrderMachine",
    initial: "cart.empty",
    states: {
      "cart.empty": {
        on: { "add-item": "cart.ready" },
      },
      "cart.ready": {
        on: {
          "add-item": "cart.ready",
          "remove-item": ["cart.ready", "cart.empty"], // conditional
          review: "cart.review",
        },
      },
      "cart.review": {
        on: {
          edit: "cart.ready",
          submit: "order.pending",
        },
      },
      "order.pending": {
        on: {
          confirm: "order.confirmed",
          cancel: "order.cancelled",
        },
      },
      "order.confirmed": {
        on: { ship: "order.shipped" },
      },
      "order.shipped": {
        on: { deliver: "order.delivered" },
      },
      "order.delivered": { terminal: true },
      "order.cancelled": { terminal: true },
    },
  },

  operations: {
    "api.create": {
      input: "OrderCreateInput",
      output: "Order",
    },
    "api.read": {
      input: "{ id: UUID }",
      output: "Order",
    },
    "page.checkout": {
      input: "{ order_id: UUID }",
      output: "HTML",
    },
    "page.confirmation": {
      input: "{ order_id: UUID }",
      output: "HTML",
    },
    "fragment.cart-summary": {
      input: "{ order_id: UUID }",
      output: "HTML",
    },
    "fragment.line-item": {
      input: "{ item_id: UUID }",
      output: "HTML",
    },
  },

  // Transitions reference states from the machine above
  transitions: {
    submit: {
      from: ["cart.ready", "cart.review"],
      to: "order.pending",
      guards: ["cart.items.length > 0", "cart.payment_method != null"],
      side_effects: ["create_order", "reserve_inventory", "send_confirmation_email"],
      output: "HTML",
    },
    confirm: {
      from: ["order.pending"],
      to: "order.confirmed",
      guards: ["payment.verified"],
      side_effects: ["charge_payment", "notify_warehouse"],
      output: "HTML",
    },
    cancel: {
      from: ["order.pending"],
      to: "order.cancelled",
      guards: [],
      side_effects: ["release_inventory", "send_cancellation_email"],
      output: "HTML",
    },
  },

  errors: [
    { code: "INVALID_STATE", when: "current_state not valid for transition" },
    { code: "GUARD_FAILED", when: "guard condition not met" },
    { code: "NOT_FOUND", when: "order does not exist" },
  ],
} as const;
```

#### `transition/submit.handler.ts`

```typescript
// features/order.v1/transition/submit.handler.ts
import { contract } from "../contract";
import { validateTransition, evaluateGuards } from "../../../generated/machines";
import { renderFragment } from "../../../core/runtime/render";
import { orderRepository } from "../../../core/db/repositories";

interface Input {
  current_state: string;
  order_id: string;
  payload: { payment_method: string };
}

interface Output {
  next_state: string;
  content: string;
}

export async function handle(input: Input): Promise<Output> {
  const transition = contract.transitions.submit;

  // 1. Validate state transition is allowed
  if (!transition.from.includes(input.current_state)) {
    return {
      next_state: input.current_state,
      content: renderFragment("error.invalid-transition", {
        current: input.current_state,
        allowed: transition.from,
      }),
    };
  }

  // 2. Evaluate guards
  const order = await orderRepository.findById(input.order_id);
  const guardResults = await evaluateGuards(order, transition.guards);

  if (!guardResults.passed) {
    return {
      next_state: input.current_state,
      content: renderFragment("error.guard-failed", {
        guard: guardResults.failed,
      }),
    };
  }

  // 3. Execute side effects
  await orderRepository.update(input.order_id, {
    status: transition.to,
    payment_method: input.payload.payment_method,
  });
  await reserveInventory(order.id);
  await sendConfirmationEmail(order);

  // 4. Return next state + rendered content
  return {
    next_state: transition.to,
    content: renderFragment("order.confirmation", { order }),
  };
}
```

#### `api/create.handler.ts`

```typescript
// features/order.v1/api/create.handler.ts
import type { OrderCreateInput, Order } from "../../../generated/types";
import { validateOrderCreateInput } from "../../../generated/validators";
import { orderRepository } from "../../../core/db/repositories";
import { contract } from "../contract";

export async function handle(input: OrderCreateInput): Promise<Order> {
  validateOrderCreateInput(input);

  // New orders start in the initial state
  const order = await orderRepository.create({
    ...input,
    status: contract.machine.initial,
  });

  return order;
}
```

#### `page/checkout.handler.ts`

```typescript
// features/order.v1/page/checkout.handler.ts
import { renderPage } from "../../../core/runtime/render";
import { orderRepository } from "../../../core/db/repositories";
import { contract } from "../contract";

export async function handle(input: { order_id: string }): Promise<string> {
  const order = await orderRepository.findById(input.order_id);

  // Determine available transitions from current state
  const currentState = contract.machine.states[order.status];
  const availableTransitions = Object.keys(currentState.on || {});

  return renderPage("order.checkout", {
    order,
    current_state: order.status,
    available_transitions: availableTransitions,
    fragments: {
      "#cart-summary": "order.v1/fragment/cart-summary",
      "#line-items": "order.v1/fragment/line-item",
    },
  });
}
```

---

### Artifact Type Reference

Within each entity folder, artifacts are organized by type:

| Subfolder     | Returns      | State Machine | Purpose                                          |
| ------------- | ------------ | ------------- | ------------------------------------------------ |
| `api/`        | JSON         | No            | Programmatic data access, external integrations  |
| `page/`       | Full HTML    | No            | Complete page renders (initial load, navigation) |
| `fragment/`   | HTML partial | No            | AJAX-swappable regions, UI components            |
| `transition/` | HTML + state | **Yes**       | User actions that change application state       |

### Handler Naming Convention

Within each subfolder, handlers follow the pattern `{action}.handler.ts`:

```
order.v1/
├── api/
│   ├── create.handler.ts     # POST /api/orders
│   ├── read.handler.ts       # GET  /api/orders/:id
│   └── list.handler.ts       # GET  /api/orders
├── page/
│   ├── checkout.handler.ts   # GET  /orders/:id/checkout
│   └── confirmation.handler.ts
├── fragment/
│   ├── cart-summary.handler.ts
│   └── line-item.handler.ts
└── transition/
    ├── submit.handler.ts     # POST /orders/:id/transition/submit
    ├── confirm.handler.ts
    └── cancel.handler.ts
```

> The contract is **authoritative**. Handlers implement exactly what the contract specifies.

---

#### `tests.ts`

- Asserts contract compliance for **all** operations in the entity
- Tests each API endpoint input/output
- Tests each page renders correctly
- Tests each fragment returns valid HTML
- Tests each transition:
  - Valid state transitions succeed
  - Invalid state transitions are rejected
  - Guards are evaluated correctly
  - Side effects are triggered
- Local and deterministic—no external dependencies

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
user.v1/
user.v2/
```

Benefits:

- AI can diff versions mechanically
- Safe backports and forward‑ports
- Deletion is explicit and auditable

---

### When to Version

Create a new version (`v2`, `v3`, etc.) when:

- Changing inputs/outputs in `contract.ts`
- Modifying state machine states or transitions
- Adding/removing guards or side effects
- Altering the fundamental behavior of any operation

Do NOT version for:

- Bug fixes that don't change the contract
- Performance optimizations
- Internal refactoring within handlers
- Code style or formatting changes

#### Edge Cases

| Scenario                               | Action                                            |
| -------------------------------------- | ------------------------------------------------- |
| Bug fix reveals contract was ambiguous | Create new version with clarified contract        |
| Fix changes observable behavior        | Create new version                                |
| Contract typo (no deployed usage)      | Fix in place, no version bump                     |
| Security patch with same I/O           | Fix in place, document in README                  |
| Deprecating a version                  | Mark contract as `deprecated: true`, keep running |
| Adding new transition to machine       | Create new version                                |
| Adding new fragment                    | Can add in place if contract allows               |

---

## Core vs Features

### `/core`

- **Never modified by AI**
- Contains invariants, policies, and lifecycles
- Small, boring, stable

Examples:

- Authorization rules
- Deployment constraints
- Domain primitives

---

### `/core/lib` (Shared Primitives)

The **one exception** to "no shared helpers": tested, contract-defined primitives that are:

1. **Pure functions** with no side effects
2. **Fully tested** with their own test suites
3. **Versioned** via semantic versioning in their module
4. **Contract-defined** with explicit input/output types

```
core/lib/
├── validators/
│   ├── email.ts          # isEmail(input: string): boolean
│   ├── email.tests.ts
│   ├── uuid.ts
│   ├── phone.ts
│   └── index.ts          # Re-exports all validators
├── formatters/
│   ├── currency.ts
│   ├── date.ts
│   └── index.ts
├── parsers/
│   ├── json-safe.ts
│   └── index.ts
└── errors/
    ├── types.ts          # AppError, ValidationError, etc.
    ├── factory.ts        # createError(), wrapError()
    └── index.ts
```

**Why this is safe**:

- Features import primitives, not business logic
- Changes to `core/lib` require human review
- Each utility is small enough to be fully understood
- AI can regenerate feature validation by composing these primitives

---

### `/core/db` (Database Layer)

All database access lives in `core/db/`, not in features. Features call repository methods; they never execute raw queries.

```
core/db/
├── client.ts             # Database connection/pool
├── migrations/           # Schema migrations (human-managed)
│   ├── 001_create_users.sql
│   └── 002_add_orders.sql
└── repositories/
    ├── user.repository.ts
    ├── order.repository.ts
    └── index.ts
```

#### Repository Pattern

```typescript
// core/db/repositories/user.repository.ts
export const userRepository = {
  create: (input: CreateUserInput): Promise<User> => { ... },
  findById: (id: string): Promise<User | null> => { ... },
  findByEmail: (email: string): Promise<User | null> => { ... },
  update: (id: string, input: UpdateUserInput): Promise<User> => { ... },
  delete: (id: string): Promise<void> => { ... },
  list: (filters: UserFilters): Promise<User[]> => { ... },
};
```

**Why repositories live in core**:

- Database schema is an invariant, not a feature
- Migrations require human oversight
- Prevents AI from writing raw SQL in features
- Single source of truth for data access patterns

---

### `/core/schema` (Canonical Schema)

The **canonical schema** is the single, authoritative specification of all data entities, constraints, and relationships in the system. All other representations—TypeScript types, validators, database migrations, API contracts—are **derived artifacts**.

```
core/schema/
├── canonical.cue          # THE source of truth for all data models
└── README.md
```

#### Why CUE?

[CUE](https://cuelang.org/) is a constraint-based configuration language designed for schema definition:

| Property                   | Benefit                                                    |
| -------------------------- | ---------------------------------------------------------- |
| **Closed by default**      | Undefined fields are rejected—no accidental data drift     |
| **Constraints as types**   | Validation rules are embedded in the schema itself         |
| **Single file legibility** | AI reads one file to understand the entire domain model    |
| **Generative**             | Produces TypeScript types, JSON Schema, OpenAPI, SQL DDL   |
| **Composable**             | Define base types, extend with domain-specific constraints |
| **Diffable**               | Schema changes are mechanically auditable                  |

#### Example: `canonical.cue`

```cue
// core/schema/canonical.cue
// THE authoritative schema for all data in this application

package schema

import "time"

// ========== Primitives ==========
#UUID: string & =~"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
#Email: string & =~"^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"
#Timestamp: string & time.Format(time.RFC3339)

// ========== Enums ==========
#UserRole: "admin" | "member" | "guest"
#OrderStatus: "pending" | "confirmed" | "shipped" | "delivered" | "cancelled"

// ========== Entities ==========
#User: {
    id:         #UUID
    email:      #Email
    role:       #UserRole
    created_at: #Timestamp
    updated_at: #Timestamp
    ...  // closed: no additional fields
}

#Order: {
    id:          #UUID
    user_id:     #UUID  // FK: #User.id
    status:      #OrderStatus
    total_cents: int & >=0
    items:       [...#OrderItem]
    created_at:  #Timestamp
    ...  // closed
}

#OrderItem: {
    product_id:  #UUID
    quantity:    int & >=1
    price_cents: int & >=0
    ...  // closed
}

// ========== Operations ==========
#UserCreateInput: {
    email:    #Email
    password: string & strings.MinRunes(12)
    role?:    #UserRole | *"member"  // optional with default
}

#UserCreateOutput: {
    user_id: #UUID
}

// ========== State Machines ==========
#MachineState: string

#CartMachine: {
    name: "CartMachine"
    initial: "cart.empty"
    states: {
        "cart.empty": {
            transitions: ["cart.add-item"]
        }
        "cart.ready": {
            transitions: ["cart.add-item", "cart.remove-item", "cart.review"]
        }
        "cart.review": {
            transitions: ["cart.edit", "order.submit"]
        }
    }
}

#OrderMachine: {
    name: "OrderMachine"
    initial: "order.pending"
    states: {
        "order.pending": {
            transitions: ["order.confirm", "order.cancel"]
        }
        "order.confirmed": {
            transitions: ["order.ship"]
        }
        "order.shipped": {
            transitions: ["order.deliver"]
        }
        "order.delivered": {
            terminal: true
        }
        "order.cancelled": {
            terminal: true
        }
    }
}

// ========== Transition Definitions ==========
#Transition: {
    name:    string
    version: int
    machine: string
    from:    [...#MachineState]
    to:      #MachineState
    guards:  [...string]
    effects: [...string]
}

#OrderSubmitV1: #Transition & {
    name:    "order.submit"
    version: 1
    machine: "OrderMachine"
    from:    ["cart.ready", "cart.review"]
    to:      "order.pending"
    guards:  ["cart.items.length > 0", "cart.payment_method != null"]
    effects: ["create_order", "reserve_inventory", "send_confirmation_email"]
}
```

#### Generation Pipeline

The canonical schema generates derived artifacts into `/generated`:

```bash
# Validate schema
cue vet core/schema/canonical.cue

# Generate TypeScript types
cue export core/schema/canonical.cue --out json | scripts/generate-types.ts

# Generate validators
cue export core/schema/canonical.cue --out json | scripts/generate-validators.ts
```

#### Schema Rules

| Constraint | Requirement                                       |
| ---------- | ------------------------------------------------- |
| Location   | `/core/schema/canonical.cue` only                 |
| Ownership  | Human-only modifications                          |
| Validation | CI validates all features against schema          |
| Generation | Types and validators regenerated on schema change |
| Versioning | Schema changes require migration plan             |

#### Schema-to-Feature Integration

The canonical schema defines **what**; features define **how**. This integration uses a hybrid model:

| Layer                 | Language   | Responsibility                                        |
| --------------------- | ---------- | ----------------------------------------------------- |
| `canonical.cue`       | CUE        | Entities, primitives, constraints, base machines      |
| `generated/*`         | TypeScript | Types, validators, machine helpers (derived from CUE) |
| `contract.ts`         | TypeScript | Entity operations, transitions, side effects          |
| `{type}/*.handler.ts` | TypeScript | Implementation                                        |

**Directory structure:**

```
core/schema/
├── canonical.cue              # Entities + primitives + base machine definitions
└── README.md

generated/
├── types.ts                   # Entity types (from CUE)
├── machines.ts                # Machine validators (from CUE)
└── validators.ts              # Runtime validators (from CUE constraints)

features/order.v1/
├── contract.ts                # Entity contract: machine + operations + transitions
├── api/
│   ├── create.handler.ts
│   └── read.handler.ts
├── page/
│   └── checkout.handler.ts
├── fragment/
│   └── cart-summary.handler.ts
├── transition/
│   ├── submit.handler.ts
│   └── cancel.handler.ts
├── tests.ts
└── README.md
```

**Step 1: Define operation signature in CUE**

```cue
// core/schema/canonical.cue (operations section)

// ========== Operations ==========
#UserCreateV1Input: {
    email:    #Email
    password: string & strings.MinRunes(12)
    role?:    #UserRole | *"member"
}

#UserCreateV1Output: {
    user_id: #UUID
}
```

**Step 2: Generate TypeScript types**

```typescript
// generated/operations.ts (auto-generated from CUE)

export interface UserCreateV1Input {
  email: Email;
  password: string;
  role?: UserRole;
}

export interface UserCreateV1Output {
  user_id: UUID;
}
```

```typescript
// generated/validators.ts (auto-generated from CUE constraints)

export function validateUserCreateV1Input(input: unknown): asserts input is UserCreateV1Input {
  // Generated from CUE constraints:
  // - email matches #Email regex
  // - password.length >= 12
  // - role is "admin" | "member" | "guest" if present
}
```

**Step 3: Feature contract binds types to runtime metadata**

```typescript
// features/user.create.v1/contract.ts
import type { UserCreateV1Input, UserCreateV1Output } from "../../generated/operations";

export interface Contract {
  name: "user.create";
  version: 1;
  input: UserCreateV1Input;
  output: UserCreateV1Output;
  side_effects: readonly ["create_user_record", "send_welcome_email"];
}

export const contract: Contract = {
  name: "user.create",
  version: 1,
  side_effects: ["create_user_record", "send_welcome_email"],
} as const;
```

**Step 4: Handler implements the contract**

```typescript
// features/user.create.v1/handler.ts
import type { UserCreateV1Input, UserCreateV1Output } from "../../generated/operations";
import { validateUserCreateV1Input } from "../../generated/validators";
import { userRepository } from "../../core/db/repositories";
import { hash } from "../../core/lib/crypto";

export async function handle(input: UserCreateV1Input): Promise<UserCreateV1Output> {
  // Validation is generated from CUE constraints
  validateUserCreateV1Input(input);

  const user = await userRepository.create({
    email: input.email,
    passwordHash: await hash(input.password),
    role: input.role ?? "member",
  });

  // Side effect: send_welcome_email would be triggered here

  return { user_id: user.id };
}
```

#### Why This Hybrid Model?

| Concern                    | Solution                                                         |
| -------------------------- | ---------------------------------------------------------------- |
| **Single source of truth** | CUE defines all types and constraints                            |
| **AI legibility**          | One file (`canonical.cue`) describes entire data domain          |
| **Type safety**            | Generated TypeScript types enforce schema at compile time        |
| **Validation**             | Generated validators enforce CUE constraints at runtime          |
| **Local reasoning**        | Feature folder contains handler + tests; types from one location |
| **No drift**               | CI validates features against schema; regenerates on change      |
| **Side effects explicit**  | Contract declares side effects (not in CUE—runtime concern)      |

#### What CUE Owns vs. What Features Own

| CUE (Canonical Schema) | Feature (Contract + Handlers) |
| ---------------------- | ----------------------------- |
| Entity definitions     | Operation declarations        |
| Primitive types        | Side effect declarations      |
| Validation constraints | Guard evaluation logic        |
| Base machine templates | Concrete machine instances    |
| Relationships (FKs)    | Transition handlers           |
|                        | Rendered content (HTML)       |
|                        | Orchestration logic           |

---

### `/features`

- Fully AI‑modifiable
- All business logic lives here
- Designed for regeneration

AI failures are _contained_.

---

### `/adapters` (Edges Only)

Adapters translate the outside world into feature calls.

**Rules**:

- No business logic
- No branching beyond IO concerns
- Thin, dumb, disposable

#### How Adapters Route to Features

Adapters route to entity feature handlers based on path patterns:

```typescript
// adapters/http/routes.ts
import * as userApi from "../../features/user.v1/api";
import * as userPage from "../../features/user.v1/page";
import * as userFragment from "../../features/user.v1/fragment";

import * as orderApi from "../../features/order.v1/api";
import * as orderPage from "../../features/order.v1/page";
import * as orderFragment from "../../features/order.v1/fragment";
import * as orderTransition from "../../features/order.v1/transition";

export const routes = [
  // === User Entity (no state machine) ===

  // API endpoints: return JSON
  { method: "POST", path: "/api/users", handler: wrapApi(userApi.create) },
  { method: "GET", path: "/api/users/:id", handler: wrapApi(userApi.read) },
  { method: "PUT", path: "/api/users/:id", handler: wrapApi(userApi.update) },
  { method: "DELETE", path: "/api/users/:id", handler: wrapApi(userApi.delete) },
  { method: "GET", path: "/api/users", handler: wrapApi(userApi.list) },

  // Page endpoints: return full HTML
  { method: "GET", path: "/users", handler: wrapPage(userPage.list) },
  { method: "GET", path: "/users/:id", handler: wrapPage(userPage.detail) },
  { method: "GET", path: "/users/:id/edit", handler: wrapPage(userPage.edit) },

  // Fragment endpoints: return HTML partials
  { method: "GET", path: "/fragments/user/:id/row", handler: wrapFragment(userFragment.row) },
  { method: "GET", path: "/fragments/user/:id/form", handler: wrapFragment(userFragment.form) },

  // === Order Entity (with state machine) ===

  // API endpoints
  { method: "POST", path: "/api/orders", handler: wrapApi(orderApi.create) },
  { method: "GET", path: "/api/orders/:id", handler: wrapApi(orderApi.read) },

  // Page endpoints
  { method: "GET", path: "/orders/:id/checkout", handler: wrapPage(orderPage.checkout) },
  { method: "GET", path: "/orders/:id/confirmation", handler: wrapPage(orderPage.confirmation) },

  // Fragment endpoints
  { method: "GET", path: "/fragments/order/:id/cart-summary", handler: wrapFragment(orderFragment.cartSummary) },
  { method: "GET", path: "/fragments/order/:id/line-item/:item_id", handler: wrapFragment(orderFragment.lineItem) },

  // Transition endpoints: receive state, return next state + HTML
  { method: "POST", path: "/orders/:id/transition/submit", handler: wrapTransition(orderTransition.submit) },
  { method: "POST", path: "/orders/:id/transition/confirm", handler: wrapTransition(orderTransition.confirm) },
  { method: "POST", path: "/orders/:id/transition/cancel", handler: wrapTransition(orderTransition.cancel) },
];

// Wrapper helpers
function wrapApi(handler) {
  return async (req, res) => {
    const result = await handler.handle(req.body);
    res.json(result);
  };
}

function wrapPage(handler) {
  return async (req, res) => {
    const html = await handler.handle({ ...req.params, ...req.query });
    res.type("html").send(html);
  };
}

function wrapFragment(handler) {
  return async (req, res) => {
    const html = await handler.handle({ ...req.params, ...req.query });
    res.type("html").send(html);
  };
}

function wrapTransition(handler) {
  return async (req, res) => {
    const result = await handler.handle({
      current_state: req.body.state,
      order_id: req.params.id,
      payload: req.body.payload,
    });
    res.set("X-Machine-State", result.next_state);
    res.type("html").send(result.content);
  };
}
```

#### Version Routing Strategies

| Strategy     | Example               | Use When                            |
| ------------ | --------------------- | ----------------------------------- |
| Path-based   | `/api/v2/orders`      | Breaking changes, long-term support |
| Header-based | `Accept-Version: v2`  | Gradual migration, same endpoint    |
| Import alias | `order.v2/api/create` | Internal routing by version         |

---

## Naming Conventions (Mechanical Symmetry)

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

## What Is Explicitly Forbidden

- `utils/` directories (use `core/lib/` instead)
- Shared business logic across features
- Implicit globals
- Framework magic
- Reflection
- Runtime behavior not declared in contracts
- Cross‑feature imports
- Circular dependencies
- Hidden state
- Aspect‑oriented programming
- Monolithic functions (>50 LOC)
- Functions with >1 argument (use input objects)
- Large feature cells (>1000 LOC)
- Complex conditionals based on environment, flags, or context
- Runtime configuration changes that alter behavior
- Raw database queries in features (use `core/db/repositories/`)

These create **non‑local reasoning requirements**, which AI handles poorly.

**Allowed in `core/lib/`**: Pure, tested, primitive utilities (validators, formatters, parsers).

---

## Entity Feature Reference

All features follow the pattern: `{entity}.v{version}/`

### Entity Structure

```
{entity}.v{version}/
├── contract.ts           # Entity contract: machine + operations + transitions
├── api/                  # JSON endpoints
│   └── {action}.handler.ts
├── page/                 # Full HTML pages
│   └── {view}.handler.ts
├── fragment/             # HTML partials
│   └── {component}.handler.ts
├── transition/           # State machine transitions (if entity has machine)
│   └── {action}.handler.ts
├── tests.ts
└── README.md
```

### Entities without State Machines

Simple CRUD entities have `api/`, `page/`, and `fragment/` subfolders only:

```
features/
├── user.v1/
│   ├── contract.ts         # machine: null
│   ├── api/
│   ├── page/
│   ├── fragment/
│   └── tests.ts
├── product.v1/
└── category.v1/
```

### Entities with State Machines

Stateful entities add a `transition/` subfolder:

```
features/
├── order.v1/
│   ├── contract.ts         # machine: { states, transitions }
│   ├── api/
│   ├── page/
│   ├── fragment/
│   ├── transition/         # State machine handlers
│   │   ├── submit.handler.ts
│   │   ├── confirm.handler.ts
│   │   └── cancel.handler.ts
│   └── tests.ts
├── cart.v1/
│   ├── contract.ts
│   ├── fragment/
│   ├── transition/
│   │   ├── add-item.handler.ts
│   │   ├── remove-item.handler.ts
│   │   └── review.handler.ts
│   └── tests.ts
└── subscription.v1/
```

---

### Layer Relationship

```
┌─────────────────────────────────────────────────────┐
│           features/{entity}.v{version}/             │
│                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐  │
│  │ transition/ │ │   page/     │ │  fragment/   │  │
│  │ State       │ │ Full HTML   │ │  HTML        │  │
│  │ machine     │ │ documents   │ │  partials    │  │
│  └──────┬──────┘ └──────┬──────┘ └──────┬───────┘  │
│         │               │               │          │
│  ┌──────┴───────────────┴───────────────┴───────┐  │
│  │                    api/                       │  │
│  │              JSON endpoints                   │  │
│  └──────────────────────┬───────────────────────┘  │
│                         │                          │
│  ┌──────────────────────┴───────────────────────┐  │
│  │              contract.ts                      │  │
│  │   Entity machine + operations + transitions   │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│              core/db/repositories/                  │
│            Data access, queries, persistence        │
├─────────────────────────────────────────────────────┤
│         core/schema/canonical.cue                   │
│   Entities, Primitives, Constraints, Base Machines  │
└─────────────────────────────────────────────────────┘
```

**Key insight**: The contract is the single source of truth for an entity. All handlers within the entity implement operations declared in the contract.

---

## Error Handling Strategy

Errors are declarative, typed, and handled consistently across the architecture.

### Error Types (in `core/lib/errors/`)

```typescript
// core/lib/errors/types.ts
export type ErrorCode =
  | "VALIDATION_ERROR"
  | "NOT_FOUND"
  | "CONFLICT"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "INTERNAL_ERROR"
  | "EXTERNAL_SERVICE_ERROR";

export interface AppError {
  code: ErrorCode;
  message: string;
  details?: Record<string, unknown>;
  cause?: Error;
}
```

### Error Declaration in Contracts

Every feature contract declares possible errors:

```typescript
// features/user.create.v1/contract.ts
export const contract = {
  name: "user.create",
  version: 1,
  inputs: { email: "string", password: "string" },
  outputs: { user_id: "uuid" },
  errors: [
    { code: "VALIDATION_ERROR", when: "Invalid email or password format" },
    { code: "CONFLICT", when: "Email already exists" },
  ],
  side_effects: ["create_user_record", "send_welcome_email"],
};
```

### Handler Error Pattern

```typescript
// features/user.create.v1/handler.ts
import { createError } from "../../core/lib/errors";

export async function handler(input: Input): Promise<Result<Output, AppError>> {
  const existing = await userRepository.findByEmail(input.email);
  if (existing) {
    return err(createError("CONFLICT", "Email already exists"));
  }
  // ... success path
  return ok({ user_id });
}
```

### Adapter Error Translation

Adapters translate `AppError` to protocol-specific responses:

```typescript
// adapters/http/error-handler.ts
const HTTP_STATUS: Record<ErrorCode, number> = {
  VALIDATION_ERROR: 400,
  NOT_FOUND: 404,
  CONFLICT: 409,
  UNAUTHORIZED: 401,
  FORBIDDEN: 403,
  INTERNAL_ERROR: 500,
  EXTERNAL_SERVICE_ERROR: 502,
};

export function handleError(error: AppError, res: Response) {
  res.status(HTTP_STATUS[error.code]).json({
    error: error.code,
    message: error.message,
    details: error.details,
  });
}
```

### Error Handling Rules

| Layer              | Responsibility                                     |
| ------------------ | -------------------------------------------------- |
| `core/lib/errors/` | Define error types, factory functions              |
| Feature handler    | Return typed errors, never throw                   |
| Feature contract   | Declare all possible error codes                   |
| Adapter            | Translate to protocol (HTTP status, CLI exit code) |
| Tests              | Verify each declared error is testable             |

---

## Change Workflow (AI‑First)

1. Human Supervisor makes a request to the Planner Agent
2. Planner Agent breaks down the request into discrete tasks and stores each in the `/tasks/todo/` directory with the following naming convention: `[priority]_[task_agent]_[short_description].md` (e.g., `001_backend-engineer_implement-user-create-handler.md`).
   - **Priority is unique**: No two tasks may share the same priority number. Priority determines execution order.
   - **Tasks are ephemeral**: Completed tasks move to `/tasks/done/` and are cleared periodically.
   - Task dependencies can be specified in YAML frontmatter within each task file.
   - All features should be implemented in degrees of complexity, starting with a minimal viable version and iterating towards the full specification.
   - The Planner Agent should ensure each task is small enough to be completed by a single task agent in one iteration and report back if the task is too large or complex.
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

### Rollback Procedure

All work is performed on **feature branches**, enabling safe rollback via Git.

#### Branch Strategy

```
main              ← Production-ready, protected
  └── workflow/add-user-search
        ├── task/001-create-contract
        ├── task/002-implement-handler
        └── task/003-add-tests
```

#### Rollback Scenarios

| Scenario                         | Action                                                             |
| -------------------------------- | ------------------------------------------------------------------ |
| Task failed mid-execution        | `git checkout .` to discard changes, move task to `/tasks/issues/` |
| Workflow completed but CI failed | `git reset --hard origin/main` on feature branch, re-plan          |
| Deployed but needs reverting     | `git revert <commit>` on main, deploy revert commit                |
| Feature version deprecated       | Keep feature directory, mark contract as `deprecated: true`        |

#### Rollback Commands

```bash
# Discard uncommitted changes in current task
git checkout .

# Reset branch to before workflow started
git reset --hard origin/main

# Revert a specific deployed commit
git revert --no-edit <commit-sha>

# Delete a feature branch after failed workflow
git branch -D workflow/failed-workflow
```

#### Immutability Principle

Once a feature version is deployed to production:

- **Never modify it in place**
- Create a new version (`v2`) for behavior changes
- Use `git revert` to undo, not `git reset`

---

## Regression Prevention

The architecture ensures regressions are detected and prevented through:

- **Immutable Versions**: Each feature version is immutable once deployed; behavior changes require new versions
- **Contract Diffing**: CI compares `contract.ts` across versions to detect breaking changes
- **Cross-Version Testing**: Tests from previous versions can be run against new versions to verify backward compatibility when required
- **Local Test Isolation**: Each feature's tests are self-contained and deterministic, ensuring failures are traceable to specific features
- **Invariant Enforcement**: CI validates that all contracts and policy invariants are upheld before deployment

---

## Mental Model Shift

**Old**:

> Architecture helps humans understand systems

**New**:

> Architecture helps machines safely change systems

Humans steward meaning.
Machines steward consistency.

---

## Design North Star

> _If an AI cannot safely modify one feature without understanding the entire codebase, the architecture has already failed._

---

## Appendix: When This Architecture Is a Bad Fit

- Highly exploratory research code
- Performance‑critical inner loops
- Novel algorithm design

In those cases, keep AI advisory, not authoritative.

---

## Agents

### Supervisor

This agent is the human overseer and prime-mover who provides high-level intent, reviews AI-generated changes, and ensures alignment with business goals and policies. It acts as the final authority on decisions and maintains accountability for the overall system.

### Product Manager

This agent is responsible for defining the vision, strategy, and roadmap for a software product. It acts as the liaison between stakeholders and the development team, using market analytics and user research to prioritize features.

- Defines product vision and strategy
- Prioritizes product features and backlog
- Collaborates with stakeholders to gather requirements
- Communicates product goals to the development team
- Monitors product performance and user feedback
- Creates high-level feature specifications
- Provides context and rationale for prioritization decisions

### Planner Agent

This agent is responsible for translating Product Manager specifications and Supervisor intent into actionable, atomic tasks. It focuses on **technical decomposition**, not product strategy.

- Receives feature specifications from Product Manager
- Breaks down features into discrete, executable tasks
- Assigns tasks to appropriate Task Agents
- Ensures tasks are small enough for single-iteration completion
- Specifies task dependencies in YAML frontmatter
- Validates that tasks align with architecture constraints
- Estimates complexity and flags tasks that are too large

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
