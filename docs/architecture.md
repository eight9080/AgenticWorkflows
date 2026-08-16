# Architecture Design Document

## Overview

**AgenticWorkflows** is a demonstration ASP.NET Core Minimal API that models a simple work-item backlog. Its primary purpose is to serve as a realistic but small target application for showcasing GitHub Copilot agentic workflows — automated agents that perform documentation drift detection, test quality review, duplicate code detection, and API error contract review.

The solution is structured as a single .NET solution (`AgenticWorkflows.slnx`) containing one API project and one xUnit test project.

---

## Major Components

### `AgenticWorkflows.Api` (ASP.NET Core Minimal API)

| Layer | Type | Responsibility |
|---|---|---|
| **Entry point** | `Program.cs` | Registers services, defines route groups, maps endpoints |
| **Models** | `WorkItem`, `WorkItemStatus`, `WorkItemSummary`, `CreateWorkItemRequest`, `ValidationError` | Immutable data records representing domain entities and API contracts |
| **Services** | `WorkItemService` | In-memory CRUD + validation + summary calculations for work items |
| **Services** | `NotificationComposer` | Stateless formatter that produces plain-text notification messages for work items |
| **Services** | `IDateProvider` / `SystemDateProvider` | Abstracts `DateOnly.FromDateTime(DateTime.Today)` to make date-sensitive logic testable |
| **Services** | `OperationResult<T>` | Discriminated-union–style result type carrying either a success value or a list of `ValidationError`s |

### `AgenticWorkflows.Api.Tests` (xUnit)

Contains focused unit tests for `WorkItemService` using a `FixedDateProvider` stub to pin `Today` to a deterministic date.

---

## Component Interactions

```
HTTP Client
    │
    ▼
Program.cs  ──route group /work-items──►  WorkItemService
                                               │
                                               ├── IDateProvider  (injected; Today abstraction)
                                               ├── List<WorkItem> (in-memory store, seeded on startup)
                                               └── OperationResult<WorkItem> (returned to endpoints)
                                                        │
                                     (on success) ──────┤
                                                        └──► Created / Ok response
                                                        │
                                     (on failure) ──────┘──► ValidationProblem (RFC 7807)

GET /{id}/notifications  ──►  WorkItemService.Find(id)  ──►  NotificationComposer (static)
```

### Data flow summary

1. **List** (`GET /work-items`) — returns all items ordered by priority (desc) then due date (asc).
2. **Create** (`POST /work-items`) — validates the request via `WorkItemService.Validate`, returns `201 Created` with the new item or `400 ValidationProblem` on failure.
3. **Get by ID** (`GET /work-items/{id}`) — returns `200 OK` with the item or `404 Not Found`.
4. **Summary** (`GET /work-items/summary`) — returns aggregate counts and a `Health` indicator (`"Healthy"` / `"Needs attention"`).
5. **Notifications** (`GET /work-items/{id}/notifications`) — returns `404` when the item is missing; otherwise returns two formatted plain-text notification strings (created + due-soon) composed by `NotificationComposer`.

---

## Design Decisions and Patterns

### In-memory store
`WorkItemService` holds a `List<WorkItem>` seeded at startup. There is no database or persistence layer — the store is intentionally simple so the repository can remain self-contained and runnable without infrastructure dependencies.

### `IDateProvider` abstraction
Date-sensitive logic (overdue detection, due-date validation, seed data) depends on `IDateProvider.Today` rather than `DateTime.Today` directly. `SystemDateProvider` is registered as a singleton in production; `FixedDateProvider` is used in tests. This follows the *clock interface* pattern and makes all date-dependent paths deterministic under test.

### `OperationResult<T>`
A lightweight result type (not exceptions) is used to communicate validation failures from the service layer to the endpoint layer. This keeps validation logic in the service and avoids leaking exception types across layers.

### Minimal API with route groups
`Program.cs` uses a `MapGroup("/work-items")` route group, keeping endpoint definitions colocated and reducing boilerplate. OpenAPI metadata is registered via `AddOpenApi` / `MapOpenApi` (development only).

### `NotificationComposer` (static class)
Notification formatting is isolated in a static helper. Both `BuildCreatedNotification` and `BuildDueSoonNotification` share logic (title normalization, description truncation, priority label formatting) via private helpers.

---

## Agentic Workflow Integration

The repository ships several GitHub Actions workflow definitions (under `.github/workflows/`) that invoke Copilot agentic workflows against this codebase:

| Workflow | Purpose |
|---|---|
| `docs-updater` | Detects documentation drift when endpoints change |
| `generate-architecture-design-doc` | Generates or updates this architecture document |
| `test-quality-checker` | Reviews test coverage and quality |
| `duplicate-code-detector` | Finds repeated patterns (e.g., notification formatting logic) |

Sample workflow prompt definitions are provided in the `samples/` directory as `.md` files.

---

## Testing Strategy

Tests live in `AgenticWorkflows.Api.Tests` and use xUnit. The test surface covers:

- **Validation** — rejects blank title, out-of-range priority, and past due dates in a single combined assertion.
- **Field trimming** — verifies that leading/trailing whitespace is stripped from title and description on creation.
- **Summary calculations** — asserts total, open, done, overdue, and high-priority counts, plus the health string, after a known sequence of creates.

A `FixedDateProvider` stub pins `Today` so seed data and validation rules produce deterministic outcomes regardless of when tests run. There are no integration or end-to-end tests; all tests exercise the service layer directly without HTTP.
