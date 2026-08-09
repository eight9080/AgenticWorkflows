# Architecture Design Document

## Overview

AgenticWorkflows is a hands-on workshop repository that teaches participants how to build and run GitHub Agentic Workflows. The repository contains:

- A minimal **ASP.NET Core Web API** (`AgenticWorkflows.Api`) that serves as the demo application for workshop exercises.
- A **test project** (`AgenticWorkflows.Api.Tests`) with xUnit tests.
- **Agentic Workflow definitions** (`.github/workflows/*.md` and `.github/aw/`) that participants create and run against the API.
- **Sample workflow prompts** (`samples/`) to illustrate common automation patterns.

The API is deliberately simple so the workshop focus stays on the agentic-workflow tooling, not the application code.

---

## Solution Structure

```
AgenticWorkflows.slnx
├── src/
│   └── AgenticWorkflows.Api/          # ASP.NET Core Minimal API
│       ├── Models/                    # Domain records and enums
│       ├── Services/                  # Business logic and utilities
│       ├── Program.cs                 # Entry point & route definitions
│       └── appsettings*.json
└── tests/
    └── AgenticWorkflows.Api.Tests/    # xUnit test project
```

---

## Major Components

### 1. `AgenticWorkflows.Api` — Minimal API

The API is built with ASP.NET Core Minimal APIs (.NET 10). All routes are declared inline in `Program.cs` under the `/work-items` route group.

| Endpoint | Method | Description |
|---|---|---|
| `/work-items` | `GET` | Returns all work items ordered by priority then due date |
| `/work-items` | `POST` | Creates a work item; returns `201 Created` or `422` validation problem |
| `/work-items/summary` | `GET` | Returns aggregate health metrics |
| `/work-items/{id}` | `GET` | Returns a single work item or `404 Not Found` |
| `/work-items/{id}/notifications` | `GET` | Returns formatted notification strings or `404 Not Found` |
| `/` | `GET` | Redirects to `/work-items` |

OpenAPI is enabled in development via `app.MapOpenApi()`.

#### Models (`src/AgenticWorkflows.Api/Models/`)

| Type | Description |
|---|---|
| `WorkItem` | Immutable record: `Id`, `Title`, `Description`, `Priority` (1–5), `Status`, `DueDate` |
| `WorkItemStatus` | Enum: `Todo`, `InProgress`, `Done` |
| `CreateWorkItemRequest` | Input record for `POST /work-items` |
| `WorkItemSummary` | Aggregate counts: total, open, done, overdue, high-priority, health label |
| `ValidationError` | `Code` + `Message` pair returned on validation failure |

#### Services (`src/AgenticWorkflows.Api/Services/`)

| Type | Description |
|---|---|
| `WorkItemService` | Singleton in-memory store; handles create, query, validation, and summary logic |
| `NotificationComposer` | Static helper; formats plain-text "created" and "due soon" notifications |
| `OperationResult<T>` | Generic result wrapper: success with value **or** failure with validation errors |
| `IDateProvider` / `SystemDateProvider` | Abstracts `DateOnly.Today` for testability |

---

## Component Interactions

```
HTTP client
    │
    ▼
Program.cs (route handlers)
    │
    ├─── WorkItemService  ◄── IDateProvider (SystemDateProvider / FixedDateProvider)
    │         │
    │         └── in-memory List<WorkItem>
    │
    └─── NotificationComposer (static, no dependencies)
```

1. Route handlers resolve `WorkItemService` from the DI container (registered as `Singleton`).
2. `WorkItemService` calls `IDateProvider.Today` to validate due dates and compute overdue counts.
3. `NotificationComposer` operates purely on a `WorkItem` value; it has no service dependencies.
4. On validation failure, the handler converts `OperationResult.Errors` into an RFC 7807 problem-detail response via `Results.ValidationProblem`.

---

## Design Decisions and Patterns

### In-memory persistence
There is no database. `WorkItemService` holds a `List<WorkItem>` seeded with four demo items on startup. Data is lost on restart. This is intentional — the workshop focuses on workflow tooling, not data persistence.

### `IDateProvider` abstraction
`DateOnly.Today` is hidden behind `IDateProvider` so tests can inject a `FixedDateProvider` with a deterministic date. This avoids flaky date-dependent assertions without introducing a mocking library.

### `OperationResult<T>` pattern
Rather than throwing exceptions for expected validation failures, `WorkItemService.Create` returns an `OperationResult<T>` that carries either a value or a collection of `ValidationError` objects. Route handlers inspect `result.Succeeded` and choose `201 Created` or `422 Unprocessable Entity` accordingly.

### 404 for missing resources
`GET /work-items/{id}` and `GET /work-items/{id}/notifications` both call `service.Find(id)` and return `Results.NotFound()` when the item does not exist.

### Static notification formatting
`NotificationComposer` is a static class with no DI involvement. The two notification formats (`BuildCreatedNotification`, `BuildDueSoonNotification`) share structural duplication — this is highlighted in the workshop as an exercise target for the duplicate-code detector agentic workflow.

---

## Agentic Workflow Definitions

Agentic Workflows are defined as Markdown files in `.github/workflows/` and managed via the `gh aw` CLI. Each workflow file describes triggers, agent instructions, and safe-output targets. Pre-built workflows in this repository include:

| File | Purpose |
|---|---|
| `docs-updater.md` | Detects documentation drift when an endpoint changes |
| `test-quality-checker.md` | Reviews test PRs for meaningful behavioral coverage |
| `duplicate-code-detector.md` | Identifies repeated notification-formatting logic |
| `generate-architecture-design-doc.md` | Generates or updates this architecture document |

Sample prompts in `samples/` provide starting points for additional workflows participants can author.

---

## Testing Strategy

Tests live in `tests/AgenticWorkflows.Api.Tests/` and use **xUnit**.

- **`WorkItemServiceTests`** — unit tests for `WorkItemService` covering input validation, field trimming, status assignment, and summary calculations. A `FixedDateProvider` pins the current date to `2026-05-31`.
- **`WeakCoverageTests`** — intentionally low-value tests included as workshop material so participants can practice identifying poor test quality using the Test Quality Checker workflow.

There are no integration or end-to-end tests; the API surface is small enough that the unit tests cover the core business rules directly.
