# Architecture: AgenticWorkflows

## Overview

AgenticWorkflows is a workshop repository used to teach participants how to build GitHub Agentic Workflows with GitHub Copilot. It ships a minimal ASP.NET Core 10 Web API (the **Work Items API**) as the hands-on application that participants modify and review during the workshop. The repository also contains a collection of prebuilt and sample agentic workflow definitions (`.md` files under `.github/workflows/` and `samples/`) that run automated agents against the codebase.

---

## Major Components

### 1. Work Items API (`src/AgenticWorkflows.Api`)

A minimal ASP.NET Core 10 Minimal API targeting `net10.0`. It manages a simple backlog of work items held entirely in memory (no database).

| File | Responsibility |
|---|---|
| `Program.cs` | Configures DI, builds the endpoint map, and starts the host. |
| `Models/WorkItem.cs` | Immutable record representing a work item (id, title, description, priority 1–5, status, due date). |
| `Models/WorkItemStatus.cs` | `Todo`, `InProgress`, `Done` enumeration. |
| `Models/WorkItemSummary.cs` | Read-model returned by the summary endpoint (counts + health label). |
| `Models/CreateWorkItemRequest.cs` | Input model for the create endpoint. |
| `Models/ValidationError.cs` | Error code + message pair used in validation results. |
| `Services/IDateProvider.cs` / `SystemDateProvider.cs` | Abstracts "today's date" to enable deterministic testing. |
| `Services/WorkItemService.cs` | In-memory store; handles list, find, create, and summary logic. Seeds four demo items on startup. |
| `Services/OperationResult<T>.cs` | Discriminated-union-style result wrapping a value or a collection of `ValidationError`s. |
| `Services/NotificationComposer.cs` | Static helper that builds plain-text "created" and "due soon" notification strings for a work item. |

#### API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Redirects to `/work-items`. |
| `GET` | `/work-items` | Returns all items ordered by priority (desc) then due date. |
| `POST` | `/work-items` | Creates a new item; returns `201 Created` or `400 ValidationProblem`. |
| `GET` | `/work-items/summary` | Returns aggregate counts and a health label. |
| `GET` | `/work-items/{id}` | Returns a single item or `404 Not Found`. |
| `GET` | `/work-items/{id}/notifications` | Returns created and due-soon notification text or `404 Not Found`. |

OpenAPI (Scalar) is exposed at `/openapi/v1.json` in the Development environment.

---

### 2. Test Project (`tests/AgenticWorkflows.Api.Tests`)

xUnit test project that exercises `WorkItemService` directly (no HTTP layer). Key design choices:

- **`FixedDateProvider`** – test-double implementation of `IDateProvider` that pins "today" to a configurable `DateOnly`, making date-sensitive tests deterministic.
- **`WorkItemServiceTests`** – focused behaviour tests for creation validation, retrieval, summary calculation, and notification text.
- **`WeakCoverageTests`** – intentionally low-value tests included as workshop fodder for the Test Quality Checker agentic workflow.

---

### 3. Agentic Workflow Definitions (`.github/workflows/` and `samples/`)

These are Markdown files that define GitHub Agentic Workflows (not traditional GitHub Actions). Each file describes a trigger, an agent prompt, and a safe output. They are the primary teaching artefacts of the repository.

| File | Purpose |
|---|---|
| `.github/workflows/docs-updater.md` | Detects documentation drift when source files change. |
| `.github/workflows/duplicate-code-detector.md` | Finds repeated code patterns across the codebase. |
| `.github/workflows/test-quality-checker.md` | Reviews test files for meaningful behaviour coverage. |
| `.github/workflows/generate-architecture-design-doc.md` | Generates or updates `docs/architecture.md` (this file). |
| `samples/api-error-contract-reviewer.md` | Sample: reviews API error contracts on PRs. |
| `samples/api-reference-generator.md` | Sample: generates API reference documentation. |
| `samples/observability-gap-finder.md` | Sample: identifies missing observability instrumentation. |
| `samples/pull-request-test-plan-reviewer.md` | Sample: reviews whether PRs include an adequate test plan. |

Lock files (`*.lock.yml`) pin workflow versions for reproducibility.

---

## Component Interactions

```
HTTP Client
    │
    ▼
Program.cs  (Minimal API route map)
    │
    ├──▶ WorkItemService  (singleton, in-memory store)
    │         │
    │         ├── IDateProvider  (injected; SystemDateProvider in prod, FixedDateProvider in tests)
    │         └── OperationResult<WorkItem>  (returned from Create)
    │
    └──▶ NotificationComposer  (static; called directly from the notifications endpoint)
```

Dependencies flow inward: endpoints depend on services, services depend on models and abstractions. There are no outbound network calls; all state is held in a `List<WorkItem>` inside `WorkItemService`.

---

## Design Decisions and Patterns

| Decision | Rationale |
|---|---|
| **Minimal API (no controllers)** | Keeps the surface area small so participants can read the entire application in a few minutes. |
| **In-memory store (singleton)** | Removes infrastructure dependencies; the app is a teaching aid, not a production system. |
| **`IDateProvider` abstraction** | Enables deterministic unit tests without mocking frameworks or system-clock manipulation. |
| **`OperationResult<T>`** | Encodes success/failure without exceptions, making validation logic explicit and testable. |
| **Static `NotificationComposer`** | Both notification types share formatting logic (truncation, priority labels, date formatting); noted as a refactoring exercise in the workshop (duplicate-code detector workflow). |
| **`WorkItemStatus` enum** | Simple three-state lifecycle (`Todo → InProgress → Done`); no transition enforcement is needed for the demo scope. |

---

## Testing Strategy

Tests target the service layer directly; no HTTP integration tests are included. The `FixedDateProvider` pattern is the key testability enabler. The `WeakCoverageTests` file is deliberately included as a teaching example of low-value tests (the Test Quality Checker workflow is expected to flag them).
