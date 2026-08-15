# Architecture Design Document

## Overview

**AgenticWorkflows** is a workshop repository that demonstrates GitHub Copilot agentic workflows. It includes a minimal ASP.NET Core 10 Web API (`AgenticWorkflows.Api`) that serves as a live, working codebase for participants to interact with using coding agents, code reviews, and custom agentic workflow automation.

The primary purpose of the API is to act as a realistic but intentionally simple target for workshop exercises — not as a production service.

---

## Major Components

### `AgenticWorkflows.Api` (ASP.NET Core Minimal API)

The sole deployable project, located in `src/AgenticWorkflows.Api/`.

#### Models (`Models/`)

| Type | Purpose |
|---|---|
| `WorkItem` | Immutable record representing a task with `Id`, `Title`, `Description`, `Priority` (1–5), `Status`, and optional `DueDate`. |
| `WorkItemStatus` | Enum: `Todo`, `InProgress`, `Done`. |
| `WorkItemSummary` | Aggregate view: total/open/done/overdue counts, high-priority-open count, and a health label. |
| `CreateWorkItemRequest` | Inbound DTO for creating a work item. |
| `ValidationError` | Carries a field `Code` and a human-readable `Message`. |

#### Services (`Services/`)

| Type | Purpose |
|---|---|
| `WorkItemService` | Core domain logic: in-memory storage, CRUD, validation, and summary computation. Seeded with four demo items on startup. |
| `OperationResult<T>` | Discriminated-union-style result type that carries either a success value or a collection of `ValidationError` instances. |
| `NotificationComposer` | Static helper that formats plain-text "created" and "due-soon" notification messages for a `WorkItem`. |
| `IDateProvider` / `SystemDateProvider` | Abstraction over `DateOnly.FromDateTime(DateTime.Today)`, injected into `WorkItemService` so tests can fix the current date. |

#### HTTP API (`Program.cs`)

Minimal API with a `/work-items` route group:

| Verb | Route | Behaviour |
|---|---|---|
| `GET` | `/work-items` | Returns all work items ordered by priority descending, then due date ascending. |
| `POST` | `/work-items` | Validates and creates a new work item; returns `201 Created` or `400 ValidationProblem`. |
| `GET` | `/work-items/summary` | Returns aggregate counts and health status. |
| `GET` | `/work-items/{id}` | Returns the work item or `404 Not Found`. |
| `GET` | `/work-items/{id}/notifications` | Returns pre-formatted created and due-soon notification strings, or `404 Not Found`. |

OpenAPI is enabled in development via `MapOpenApi()`.

---

## Component Interactions

```
HTTP Client
    │
    ▼
Program.cs (Minimal API route handlers)
    │
    ├──► WorkItemService
    │         │
    │         ├── IDateProvider  (date abstraction)
    │         └── in-memory List<WorkItem>
    │
    └──► NotificationComposer  (static, stateless)
```

`WorkItemService` is registered as a **singleton**, so all in-memory state is shared across requests for the lifetime of the process. There is no database or external persistence.

---

## Notable Design Decisions

| Decision | Rationale |
|---|---|
| **In-memory singleton storage** | Keeps the demo self-contained with zero infrastructure dependencies. Data resets on each process restart. |
| **`OperationResult<T>` pattern** | Avoids exception-driven validation flow; lets the API layer translate domain failures into `ValidationProblem` responses cleanly. |
| **`IDateProvider` abstraction** | Allows unit tests to inject a fixed date (`FixedDateProvider`) without mocking frameworks, making time-sensitive validation deterministic. |
| **Static `NotificationComposer`** | Notification formatting is pure and stateless; no dependency injection needed. Identified in workshop exercises as a candidate for duplicate-code detection. |
| **Minimal API style** | Reduces ceremony for a workshop setting; all routes are in a single `Program.cs` file. |
| **Seed data** | Four pre-populated work items with relative due dates (relative to `IDateProvider.Today`) are created at startup to give participants immediate data to explore. |

---

## Testing Strategy

Tests are in `tests/AgenticWorkflows.Api.Tests/` using **xUnit**.

- `WorkItemServiceTests` covers the service layer directly (no HTTP layer), using `FixedDateProvider` to anchor dates.
- Key scenarios tested: multi-field validation rejection, field trimming on create, and summary aggregation counts and health label.
- `WeakCoverageTests` contains a placeholder `Assert_True` test — this is intentional workshop material used to demonstrate low-value tests.

No integration or end-to-end HTTP tests are present; the workshop relies on the `.http` file (`AgenticWorkflows.Api.http`) for manual endpoint verification.

---

## Repository Layout

```
src/
  AgenticWorkflows.Api/          # ASP.NET Core Minimal API
    Models/                      # Domain records and enums
    Services/                    # Business logic and abstractions
    Program.cs                   # DI registration and route definitions
tests/
  AgenticWorkflows.Api.Tests/    # xUnit unit tests
.github/
  workflows/                     # CI and agentic workflow definitions
  agents/                        # Agent prompt definitions
samples/                         # Sample workflow prompt files
docs/                            # Architecture documentation (this file)
```
