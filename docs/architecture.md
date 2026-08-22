# Architecture Design Document

## Overview

`AgenticWorkflows` is a workshop repository that teaches participants how to build GitHub Agentic Workflows using GitHub Copilot. It ships a small, self-contained ASP.NET Core Minimal API (`AgenticWorkflows.Api`) as the subject application on which agentic workflows operate — reviewing tests, detecting duplicate code, generating documentation, and so on.

The repository is intentionally simple so that attendees can focus on workflow authoring rather than understanding a complex codebase.

---

## Major Components

### `src/AgenticWorkflows.Api` — the subject API

An ASP.NET Core 10 Minimal API that manages a lightweight in-memory work-item backlog.

| Layer | Files | Responsibility |
|---|---|---|
| **Entry point** | `Program.cs` | Registers services, maps routes, starts the host. |
| **Models** | `Models/` | Immutable C# records and enums (`WorkItem`, `WorkItemStatus`, `CreateWorkItemRequest`, `WorkItemSummary`, `ValidationError`). |
| **Services** | `Services/WorkItemService.cs` | Business logic: CRUD, validation, summary computation, in-memory seeded data. |
| **Services** | `Services/NotificationComposer.cs` | Builds plain-text notification messages for "created" and "due soon" events. |
| **Services** | `Services/OperationResult.cs` | Generic result type that carries a value or a collection of validation errors. |
| **Services** | `Services/IDateProvider.cs` / `SystemDateProvider.cs` | Abstraction over `DateOnly.FromDateTime(DateTime.Today)` to enable deterministic testing. |

### `tests/AgenticWorkflows.Api.Tests` — xUnit test project

| File | Coverage |
|---|---|
| `WorkItemServiceTests.cs` | `WorkItemService`: validation, field trimming, status assignment, summary counts. |
| `WeakCoverageTests.cs` | Placeholder / weak tests used as an intentional workshop target for the Test Quality Checker workflow. |
| `FixedDateProvider.cs` | Test double that returns a pinned date, removing time-dependency from tests. |

### `.github/workflows/` — Agentic Workflow definitions

Pre-built and participant-authored workflows that run GitHub Copilot coding-agent tasks against the API:

| Workflow | Purpose |
|---|---|
| `test-quality-checker` | Reviews pull requests that change `tests/**` and assesses test quality. |
| `docs-updater` | Detects documentation drift when source code changes. |
| `duplicate-code-detector` | Finds repeated patterns (e.g., `NotificationComposer`) and suggests refactors. |
| `generate-architecture-design-doc` | Generates or updates this document. |
| `ci.yml` | Standard .NET build + xUnit test run on every push/PR. |

### `samples/` — Workflow prompt starters

Markdown files with pre-written workflow prompts that participants can copy and adapt (`api-error-contract-reviewer.md`, `api-reference-generator.md`, `observability-gap-finder.md`, `pull-request-test-plan-reviewer.md`).

---

## Component Interactions

```
HTTP request
    │
    ▼
Program.cs  (Minimal API route group /work-items)
    │
    ├─── WorkItemService  ──► IDateProvider  (today's date for validation & seeding)
    │         │
    │         └─► OperationResult<WorkItem>  (success / validation errors)
    │
    └─── NotificationComposer  (stateless; formats plain-text messages from WorkItem)
```

All state lives in a `List<WorkItem>` inside the singleton `WorkItemService`. There is no database or external dependency; the service is seeded with four demo work items on startup.

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Redirects to `/work-items`. |
| `GET` | `/work-items` | Returns all work items sorted by priority (desc) then due date (asc). |
| `POST` | `/work-items` | Creates a work item; returns `201 Created` or `422 Unprocessable Entity` (validation problem). |
| `GET` | `/work-items/summary` | Returns aggregate counts and a health indicator. |
| `GET` | `/work-items/{id}` | Returns a single work item or `404 Not Found`. |
| `GET` | `/work-items/{id}/notifications` | Returns pre-formatted "created" and "due-soon" notification strings, or `404 Not Found`. |

OpenAPI (`/openapi/v1.json`) is available in the Development environment.

---

## Design Decisions and Patterns

**Minimal API over Controllers** — The codebase deliberately keeps ceremony low. All routing logic sits in `Program.cs` so workshop participants can read the whole API in one file.

**In-memory singleton store** — No database is needed for a demo application. `WorkItemService` is registered as a singleton holding a `List<WorkItem>`. This makes the app stateful per process restart, which is intentional for the workshop context.

**`IDateProvider` abstraction** — Rather than calling `DateTime.Today` directly, the service depends on `IDateProvider`. `SystemDateProvider` provides the real date in production; `FixedDateProvider` pins the date in tests. This makes validation logic (past due-date check) and seeding deterministic.

**`OperationResult<T>`** — A lightweight discriminated-union–style record replaces exception-based validation. Callers inspect `Succeeded` and `Errors` instead of catching exceptions, keeping the flow explicit and easy to test.

**`NotificationComposer` duplication (intentional)** — `BuildCreatedNotification` and `BuildDueSoonNotification` share near-identical formatting logic. This is a deliberate workshop artifact for the `duplicate-code-detector` agentic workflow to discover.

---

## Testing Strategy

Tests use **xUnit** and are located in `tests/AgenticWorkflows.Api.Tests`. The project follows an arrange-act-assert style with no test framework beyond xUnit.

- `WorkItemService` is tested directly (no HTTP layer), using `FixedDateProvider` for determinism.
- There are intentionally low-value tests in `WeakCoverageTests.cs` (e.g., `Assert.True(true)`) so the Test Quality Checker workflow has something to flag.
- The CI workflow (`ci.yml`) runs `dotnet test` on every push and pull request.
