# Architecture Design Document

## Overview

**AgenticWorkflows** is a demo ASP.NET Core Minimal API (targeting .NET 10) that showcases a simple work-item tracking back-end. Its primary purpose is to serve as a realistic reference application for demonstrating GitHub Copilot agentic workflows — including automated documentation updates, duplicate-code detection, test-quality checking, and architecture document generation.

The application is intentionally small: a single project with in-memory state, designed to make workflow outcomes easy to observe and discuss.

---

## Project Structure

```
AgenticWorkflows/
├── src/
│   └── AgenticWorkflows.Api/          # ASP.NET Core Minimal API
│       ├── Models/                    # Immutable record types and enums
│       ├── Services/                  # Business logic and abstractions
│       └── Program.cs                 # App bootstrap and route registration
└── tests/
    └── AgenticWorkflows.Api.Tests/    # xUnit unit tests
```

**Solution file**: `AgenticWorkflows.slnx`

---

## Major Components

### Models (`src/AgenticWorkflows.Api/Models/`)

| Type | Kind | Responsibility |
|---|---|---|
| `WorkItem` | `record` | Immutable representation of a work item (id, title, description, priority 1–5, status, optional due date). |
| `WorkItemStatus` | `enum` | Three-state lifecycle: `Todo`, `InProgress`, `Done`. |
| `CreateWorkItemRequest` | `record` | Input DTO for creating a work item. |
| `WorkItemSummary` | `record` | Aggregated statistics returned by the summary endpoint. |
| `ValidationError` | `record` | Carries a field code and human-readable message for validation failures. |

### Services (`src/AgenticWorkflows.Api/Services/`)

| Type | Kind | Responsibility |
|---|---|---|
| `IDateProvider` / `SystemDateProvider` | Interface + impl | Abstracts `DateOnly.Today` to allow deterministic testing without system-clock dependency. |
| `WorkItemService` | Singleton service | In-memory store and sole business-logic host: list, find, create, validate, and summarise work items. Seeded with four demo items on startup. |
| `OperationResult<T>` | Generic record | Discriminated-union-style result type — carries either a successful value or a collection of `ValidationError` objects, avoiding exception-based flow control for validation. |
| `NotificationComposer` | Static class | Pure formatting logic that builds plain-text notification strings for "created" and "due soon" events from a `WorkItem`. |

### API Surface (`src/AgenticWorkflows.Api/Program.cs`)

All routes are registered via `MapGroup("/work-items")`:

| Method | Path | Description |
|---|---|---|
| `GET` | `/work-items` | Returns all work items ordered by priority descending, then due date ascending. |
| `POST` | `/work-items` | Creates a work item; returns `201 Created` or `400 ValidationProblem`. |
| `GET` | `/work-items/summary` | Returns aggregate statistics and a health indicator. |
| `GET` | `/work-items/{id:guid}` | Returns a single work item or `404 Not Found`. |
| `GET` | `/work-items/{id:guid}/notifications` | Returns pre-formatted "created" and "due soon" notification strings for a work item, or `404 Not Found`. |

A root `GET /` redirects to `/work-items`.

OpenAPI metadata is exposed at `/openapi/v1.json` in the Development environment.

---

## Component Interactions

```
HTTP Request
     │
     ▼
Program.cs (Minimal API route handlers)
     │  injects
     ▼
WorkItemService (singleton, in-memory state)
     │  uses
     ├──► IDateProvider  (clock abstraction)
     └──► OperationResult<WorkItem>  (returned to route handler)
                │
                ▼
         Route handler maps result → HTTP response
                │  (NotificationComposer used inline for /notifications)
                ▼
             HTTP Response
```

Data flows in one direction: route handlers call `WorkItemService`, which uses `IDateProvider` and returns `OperationResult<T>` or domain objects. `NotificationComposer` is a stateless helper called directly by route handlers.

---

## Design Decisions and Patterns

### In-Memory Storage
The service holds a `List<WorkItem>` seeded at startup. There is no database or persistence layer. This is intentional for a demo application — it keeps the surface area small and removes infrastructure dependencies.

### `OperationResult<T>` over Exceptions
Validation failures return a typed `OperationResult<WorkItem>.Failure(errors)` rather than throwing exceptions. Route handlers inspect `result.Succeeded` and map to `Results.ValidationProblem(...)` or `Results.Created(...)`. This pattern makes the happy-path/error-path split explicit at the call site.

### `IDateProvider` Abstraction
The current date is injected via `IDateProvider` rather than read directly from `DateTime.Today`. This is the only seam needed for deterministic tests — `FixedDateProvider` in the test project pins the date without any mocking framework.

### Minimal API
The application uses ASP.NET Core Minimal APIs (no controllers) registered as a route group. This keeps the entry point concise and co-locates route definitions with the bootstrap code.

### Immutable Records
All domain and DTO types are C# `record` types (`sealed record`), enforcing value semantics and discouraging mutation outside of the service.

---

## Testing Strategy

Tests live in `tests/AgenticWorkflows.Api.Tests` and use **xUnit** without any mocking framework.

- **`WorkItemServiceTests`** — behaviour-focused tests covering validation rejection (blank title, out-of-range priority, past due date), field trimming on creation, and summary aggregation logic.
- **`WeakCoverageTests`** — intentionally thin tests included as demo fodder for the test-quality checker agentic workflow, illustrating the contrast between meaningful behaviour tests and low-value implementation tests.
- **`FixedDateProvider`** — a test-local `IDateProvider` implementation that pins `Today` to a known date, enabling fully deterministic assertions.

All tests exercise `WorkItemService` directly; there are no HTTP-level integration tests.
