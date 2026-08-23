# Architecture Design Document

## Overview

**AgenticWorkflows** is a workshop repository that teaches participants how to build and run [GitHub Agentic Workflows](https://docs.github.com/en/copilot). It ships a small, intentionally simple ASP.NET Core 10 Minimal API — a work-item tracker — that serves as a realistic but low-complexity target for automated agentic workflows such as a test-quality checker, a duplicate-code detector, and a documentation updater.

The purpose of the API is **not** to be a production system; it exists to give agentic workflows a concrete codebase to analyse and improve during the workshop.

---

## Solution layout

```
AgenticWorkflows.slnx
├── src/
│   └── AgenticWorkflows.Api/         # ASP.NET Core 10 Minimal API
│       ├── Models/                   # Immutable record types and enums
│       ├── Services/                 # Business logic and helpers
│       └── Program.cs                # Endpoint registration and DI wiring
└── tests/
    └── AgenticWorkflows.Api.Tests/   # xUnit unit-test project
```

---

## Components

### `AgenticWorkflows.Api` — Minimal API

The API is a single-project ASP.NET Core application that uses the Minimal API model (no controllers). All endpoints are registered directly in `Program.cs`.

#### Models (`src/AgenticWorkflows.Api/Models/`)

| Type | Role |
|---|---|
| `WorkItem` | Immutable record: `Id`, `Title`, `Description`, `Priority` (1–5), `Status`, `DueDate`. |
| `WorkItemStatus` | Enum: `Todo`, `InProgress`, `Done`. |
| `WorkItemSummary` | Read-only aggregate: totals, overdue count, health label. |
| `CreateWorkItemRequest` | Input DTO for the POST endpoint. |
| `ValidationError` | Structured error with `Code` and `Message`. |

#### Services (`src/AgenticWorkflows.Api/Services/`)

| Type | Role |
|---|---|
| `WorkItemService` | Singleton. Owns an in-memory `List<WorkItem>`, seeded with four items on startup. Provides `GetAll`, `Find`, `Create`, and `GetSummary`. Validates input and returns an `OperationResult<T>`. |
| `OperationResult<T>` | Discriminated-union-style record: `Succeeded` is `true` when `Errors` is empty; carries the typed `Value` on success. |
| `IDateProvider` / `SystemDateProvider` | Abstraction over `DateOnly.Today` injected into `WorkItemService` so tests can pin the current date. |
| `NotificationComposer` | Static helper that formats plain-text "created" and "due-soon" notification strings for a given `WorkItem`. |

#### Endpoints (`Program.cs`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Redirects to `/work-items`. |
| `GET` | `/work-items` | Returns all items ordered by priority (desc), then due date (asc). |
| `POST` | `/work-items` | Creates a new item; returns `201 Created` or `400 ValidationProblem`. |
| `GET` | `/work-items/summary` | Returns a `WorkItemSummary` aggregate. |
| `GET` | `/work-items/{id:guid}` | Returns a single item or `404 Not Found`. |
| `GET` | `/work-items/{id:guid}/notifications` | Returns formatted notification strings or `404 Not Found`. |

OpenAPI document is served at `/openapi/v1.json` in the Development environment.

---

## Component interactions

```
HTTP Client
    │
    ▼
Program.cs (Minimal API endpoints)
    │
    ├─── WorkItemService  ◄── IDateProvider (SystemDateProvider / FixedDateProvider)
    │        │
    │        └── OperationResult<WorkItem>  (returned from Create)
    │
    └─── NotificationComposer  (static, called directly from the notifications endpoint)
```

All state is held in-process in `WorkItemService`. There is no database, message broker, or external dependency at runtime.

---

## Design decisions and patterns

### In-memory singleton store
`WorkItemService` is registered as a singleton and holds the work-item list in a plain `List<T>`. This keeps the project dependency-free and easy to inspect, which is the primary goal for workshop demonstrations.

### `OperationResult<T>` for validated writes
Rather than throwing exceptions or returning `null` on validation failure, `Create` returns an `OperationResult<WorkItem>`. The endpoint maps `Succeeded == false` to a standard `ValidationProblem` (RFC 7807) response, and `Succeeded == true` to `201 Created` with a `Location` header.

### `IDateProvider` for testability
Abstracting `DateOnly.Today` behind `IDateProvider` lets unit tests pin the current date via `FixedDateProvider` without any mocking framework. This is intentionally minimal — it demonstrates the pattern without adding framework overhead.

### `NotificationComposer` as a static helper
Notification formatting has no external dependencies and no mutable state, so it is implemented as a static class rather than being injected. The seeded work items include a "Refactor duplicate notifications" task, which exists specifically to give the duplicate-code-detector workflow a realistic target.

### Missing persistence (intentional)
All work-item state is lost on restart. Participants are not expected to add a database; the workshop scenarios focus on workflow and agent behaviour rather than persistence concerns.

---

## Agentic workflows

The repository ships several GitHub Agentic Workflows (under `.github/workflows/`) that target this API:

| Workflow | Purpose |
|---|---|
| `test-quality-checker` | Reviews pull requests that change `tests/**` and flags low-value tests. |
| `duplicate-code-detector` | Scans for repeated code patterns and suggests consolidation. |
| `docs-updater` | Detects drift between source changes and documentation. |
| `generate-architecture-design-doc` | Generates or updates this file. |

Workflow definitions and their lock files live alongside `.md` configuration files in `.github/workflows/`. Sample prompt definitions are in `samples/`.

---

## Testing strategy

The test project (`tests/AgenticWorkflows.Api.Tests`) uses **xUnit** and targets `net10.0`. Tests exercise `WorkItemService` directly (unit tests), using `FixedDateProvider` to control date-sensitive logic. There are no integration or end-to-end tests; the API surface is intentionally narrow and the workshop encourages participants to add meaningful behavior tests via the coding agent.

Current meaningful test coverage:
- Validation: blank title, out-of-range priority, past due date.
- Field trimming on creation.
- `GetSummary` aggregate counts and health label.
