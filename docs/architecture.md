# Architecture

## Overview

AgenticWorkflows is a workshop template repository used to teach GitHub Copilot agentic workflows. It ships a small ASP.NET Core Minimal API (`AgenticWorkflows.Api`) that tracks work items, serving as a realistic but deliberately simple target application for AI-assisted workflows such as test-quality checking, documentation updating, and duplicate-code detection.

---

## Solution structure

```
AgenticWorkflows.slnx
├── src/
│   └── AgenticWorkflows.Api/          # Minimal API host
│       ├── Models/                    # Immutable record types
│       ├── Services/                  # Business logic and helpers
│       └── Program.cs                 # Endpoint registration
├── tests/
│   └── AgenticWorkflows.Api.Tests/    # xUnit unit tests
└── samples/                           # Agentic-workflow prompt samples
```

---

## Major components

### `AgenticWorkflows.Api` (ASP.NET Core Minimal API)

The only deployable project. All HTTP endpoints are registered in `Program.cs` using the Minimal API fluent style. There is no controller layer.

#### Models (`Models/`)

| Type | Role |
|---|---|
| `WorkItem` | Immutable record representing a single work item (id, title, description, priority 1–5, status, optional due date). |
| `WorkItemStatus` | Enum: `Todo`, `InProgress`, `Done`. |
| `CreateWorkItemRequest` | Input record for the `POST /work-items` endpoint. |
| `WorkItemSummary` | Read-only aggregate returned by `GET /work-items/summary`. |
| `ValidationError` | Code + message pair used in validation failure responses. |

#### Services (`Services/`)

| Type | Role |
|---|---|
| `WorkItemService` | Singleton that owns the in-memory list of `WorkItem` records. Handles listing, lookup, creation (with validation), and summary aggregation. |
| `NotificationComposer` | Static helper that builds plain-text "created" and "due-soon" notification strings for a given `WorkItem`. |
| `OperationResult<T>` | Generic discriminated union: carries either a success value or a collection of `ValidationError`s. |
| `IDateProvider` / `SystemDateProvider` | Abstraction over `DateOnly.FromDateTime(DateTime.Today)` so tests can inject a fixed date. |

---

## HTTP API

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Redirects to `/work-items`. |
| `GET` | `/work-items` | Returns all items ordered by priority (desc) then due date (asc). |
| `POST` | `/work-items` | Creates a new item; returns `201 Created` or `400 ValidationProblem`. |
| `GET` | `/work-items/summary` | Returns aggregate health metrics. |
| `GET` | `/work-items/{id}` | Returns a single item or `404 Not Found`. |
| `GET` | `/work-items/{id}/notifications` | Returns pre-formatted "created" and "due-soon" notification strings, or `404 Not Found`. |

OpenAPI metadata is enabled in development via `MapOpenApi()`.

---

## Data flow

```
HTTP request
    │
    ▼
Program.cs (endpoint lambda)
    │  calls
    ▼
WorkItemService  ──────────────────────►  IDateProvider
    │  returns OperationResult<T>           (today's date for validation / seeding)
    ▼
HTTP response (Results.Ok / Created / NotFound / ValidationProblem)
```

Notification text is generated on-demand by `NotificationComposer` when the notifications endpoint is called; it is not persisted.

---

## Design decisions and trade-offs

| Decision | Rationale |
|---|---|
| **In-memory storage** | Keeps the demo dependency-free. Data is seeded on startup and lost on restart—intentional for a workshop context. |
| **Singleton `WorkItemService`** | Appropriate for a single-process demo without concurrency concerns. |
| **`OperationResult<T>` pattern** | Avoids exceptions for expected validation failures; makes success/failure explicit at the call site. |
| **`IDateProvider` abstraction** | Enables deterministic unit tests without mocking frameworks by using a simple `FixedDateProvider` in tests. |
| **Static `NotificationComposer`** | Both notification methods share significant structure (see duplicate-code detector sample); extracted as a static class for simplicity. |
| **Minimal API (no controllers)** | Reduces ceremony for a small demo surface; all routing is visible in a single `Program.cs` file. |

---

## Testing strategy

Tests are in `AgenticWorkflows.Api.Tests` (xUnit). Coverage targets the `WorkItemService` business logic directly, bypassing HTTP:

- **Validation**: rejects blank titles, out-of-range priorities, and past due dates.
- **Happy path creation**: verifies field trimming and status defaults.
- **Summary aggregation**: verifies counts for open, done, overdue, and high-priority items, and the health label.

A `FixedDateProvider` injects a stable `DateOnly` so test outcomes do not depend on the wall clock.

---

## Agentic-workflow samples

The `samples/` directory contains prompt files that serve as starter definitions for GitHub Agentic Workflows exercised during the workshop:

| File | Purpose |
|---|---|
| `api-error-contract-reviewer.md` | Reviews API responses for correct error contracts. |
| `api-reference-generator.md` | Generates API reference documentation. |
| `observability-gap-finder.md` | Identifies missing observability instrumentation. |
| `pull-request-test-plan-reviewer.md` | Reviews pull requests for test coverage and quality. |

These samples, together with the prebuilt workflows in `.github/workflows/`, form the primary teaching material of the repository.
