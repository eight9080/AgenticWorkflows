# Architecture Design Document

## Overview

**AgenticWorkflows** is an ASP.NET Core Minimal API that serves as a demonstration application for a GitHub Agentic Workflows workshop. It manages a simple work-item backlog and is intentionally kept small so that participants can focus on learning how to build, run, and refine GitHub Agentic Workflows (automated AI-driven workflows using `gh aw`) rather than on application complexity.

The application exposes a REST API for creating and querying work items. It is paired with a suite of GitHub Actions–based agentic workflows that act as automated reviewers, documentation generators, and code-quality checkers operating on pull requests against this repository.

---

## Major Components

### `AgenticWorkflows.Api` — ASP.NET Core Minimal API

Located in `src/AgenticWorkflows.Api/`.

| Layer | Files | Responsibility |
|---|---|---|
| **Entry point / routing** | `Program.cs` | Configures the DI container, registers services, and declares all HTTP endpoints using Minimal API route groups. |
| **Models** | `Models/` | Immutable record types (`WorkItem`, `CreateWorkItemRequest`, `WorkItemSummary`, `ValidationError`) and the `WorkItemStatus` enum. |
| **Services** | `Services/` | Business logic (`WorkItemService`), notification text generation (`NotificationComposer`), a date abstraction (`IDateProvider`/`SystemDateProvider`), and a generic result wrapper (`OperationResult<T>`). |

### `AgenticWorkflows.Api.Tests` — xUnit Test Project

Located in `tests/AgenticWorkflows.Api.Tests/`.

Unit tests for `WorkItemService` using a `FixedDateProvider` to control the current date deterministically. Tests cover validation, field trimming, and summary calculations.

### GitHub Agentic Workflows

Located in `.github/workflows/`.

| Workflow | Description |
|---|---|
| `ci.yml` | Builds and tests the .NET solution on every push/PR. |
| `generate-architecture-design-doc.lock.yml` | Agentic workflow that generates or updates `docs/architecture.md`. |
| `docs-updater.lock.yml` | Agentic workflow that detects documentation drift after endpoint changes. |
| `test-quality-checker.lock.yml` | Agentic workflow that reviews test pull requests for quality. |
| `duplicate-code-detector.md` | Sample agentic workflow definition detecting repeated notification formatting. |

---

## Component Interactions

```
HTTP Client
    │
    ▼
Program.cs  ──── Route group /work-items
    │
    ├── GET  /work-items             → WorkItemService.GetAll()
    ├── POST /work-items             → WorkItemService.Create(request)
    ├── GET  /work-items/summary     → WorkItemService.GetSummary()
    ├── GET  /work-items/{id}        → WorkItemService.Find(id)
    └── GET  /work-items/{id}/notifications → NotificationComposer.Build*Notification(item)
                                                   (via WorkItemService.Find)

WorkItemService
    ├── depends on IDateProvider (injected)
    │       └── SystemDateProvider  (production)
    │       └── FixedDateProvider   (tests)
    └── returns OperationResult<WorkItem> from Create()
```

All state is held **in-memory** (`List<WorkItem>` inside `WorkItemService`, registered as a singleton). The service seeds four example items on startup relative to the current date.

---

## Design Decisions and Patterns

### Minimal API with Route Groups
Endpoints are declared inline in `Program.cs` using `MapGroup("/work-items")`. This keeps the surface area small and eliminates controller boilerplate, which is appropriate for a workshop demo.

### In-Memory Storage (Singleton Service)
`WorkItemService` is registered as a singleton and stores items in a `List<T>`. There is no database or persistence layer. This simplifies setup for workshop participants and keeps the demo self-contained.

### `OperationResult<T>` (Result Pattern)
`WorkItemService.Create` returns `OperationResult<WorkItem>` instead of throwing exceptions. This makes validation failures explicit and easy to map to `400 ValidationProblem` responses in the routing layer.

### `IDateProvider` Abstraction
The current date is injected via `IDateProvider` rather than calling `DateOnly.FromDateTime(DateTime.Today)` directly. This makes `WorkItemService` fully testable with a fixed date (`FixedDateProvider`) without any mocking framework.

### `NotificationComposer` (Static Helper)
Notification text assembly is separated into a static class. Both `BuildCreatedNotification` and `BuildDueSoonNotification` share private helpers (`FormatPriority`, `NormalizeTitle`), though the two methods contain duplicated structure — an intentional design smell used in the duplicate-code-detector workshop exercise.

### `404 Not Found` for Missing Resources
`GET /work-items/{id}` and `GET /work-items/{id}/notifications` return `404 Not Found` when no item matches the provided GUID, rather than `200 OK` with a null body.

---

## Testing Strategy

The test project (`AgenticWorkflows.Api.Tests`) uses **xUnit** and focuses on unit-testing `WorkItemService` in isolation:

- **`FixedDateProvider`** pins `Today` to a known date so that seed items and due-date validation are deterministic.
- Tests verify behavior (validation rejects invalid input, fields are trimmed, summary counts are correct) rather than implementation details.
- There are no integration tests; the API routing layer is not tested directly.

The `test-quality-checker` agentic workflow reviews test pull requests against this strategy and flags low-value tests (e.g., `Assert_True`) as part of the workshop experience.
