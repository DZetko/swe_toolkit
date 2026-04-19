author: Daniel Zikmund, zikmund.d@gmail.com
version: 1.0.0

# Backend Architecture Rules

Reusable rules for any ASP.NET backend built on **Domain Driven Design + Clean Architecture**. These are _non-negotiable_ unless explicitly overridden in a project's own spec.

## 1. Solution layout

```
Check the project layout of files to determine where you should be placing new files
```

- One class-library project per layer.
- `Api` is the only executable host. It references `Application` and `Infrastructure`.
- Composition root (DI registration) lives in `Api/Program.cs`. Each layer exposes an `AddXxx(IServiceCollection)` extension for the layer's registrations.

## 3. Dependency direction (Clean Architecture)

```
Api  →  Application  →  Domain
Infrastructure  →  Application  →  Domain
```

- `Domain` has **zero** external dependencies. No ASP.NET, no Mongo driver, no MediatR.
- `Application` depends only on `Domain` and cross-cutting libs (FluentValidation). It **never** references Mongo, HTTP, or any concrete infrastructure.
- `Infrastructure` implements `Application` abstractions. It's the only layer that talks to external systems.
- `Api` wires things together and exposes HTTP endpoints.
- A violation of dependency direction is a build-time failure. Use `NetArchTest` in a test project to enforce it.

## 4. Domain layer rules

- Every aggregate root is a class with:
    - Private setters / `init` properties
    - Constructors / factory methods that enforce invariants
    - Behavior as methods on the aggregate — no anemic models
- Aggregates protect their own invariants. Application handlers never set raw properties on aggregates.
- Value objects are immutable record types with structural equality.
- Identifiers are strongly-typed (`PipelineId`, not `Guid`) via readonly struct or record wrappers.
- Enums with behavior become **smart enums** (a sealed class with static instances and methods), not `enum` keywords.
- Domain events inherit from a common `IDomainEvent` marker. Aggregates raise them via a protected `Raise(...)` and expose `DomainEvents` read-only.
- Domain services only when the logic doesn't naturally belong to a single aggregate.
- Repository interfaces live in Domain (`IXxxRepository`). They accept/return domain objects only — never persistence-layer types.
- Domain operations use the Result pattern to notify the caller if the operation succeeded. The result can be void (not generic) param, a primitive or a simple record
- No `DateTime.Now`, no `Guid.NewGuid()`, no `Random` inside Domain — take these as dependencies via interfaces (`IClock`, `IIdGenerator`).

## 5. Application layer rules

- **CQRS pattern.** Every use case is a `Command` or `Query` with a dedicated handler class. Handlers are injected directly into controllers as services — no in-process message bus.
- Commands are verbs (`CreatePipelineCommand`), queries start with `Get` (`GetPipelinesQuery`). One command/query per use case — no catch-all `ManageXxx` handlers.
- Handler interfaces follow a uniform shape: `ICommandHandler<TCommand, TResult>` / `IQueryHandler<TQuery, TResult>`. Each handler is a class with a single `HandleAsync` method.
- Handlers orchestrate: load aggregates, call aggregate methods, persist, dispatch events. They must not contain business rules that belong inside an aggregate.
- DTOs are records, one set per aggregate (`XxxDto`). Mapping via hand-written extension methods (`ToDto()`) — no AutoMapper.
- Validators live alongside the command/query they validate. Validation is invoked by the handler at the top of `HandleAsync`, or by a thin decorator/wrapper around handlers — never scattered across the controller.
- Cross-cutting abstractions (`IClock`, `INotificationService`, `IPipelineExecutionEngine`, etc.) are declared in Application and implemented in Infrastructure.
- Handlers never throw raw `Exception` — always a typed exception from Domain or a specific Application-level exception (`NotFoundException`, `ConflictException`).
- Transactions: rely on aggregate-boundary atomicity (one aggregate per transaction). Cross-aggregate consistency goes through domain events, not distributed transactions.

## 6. Infrastructure layer rules

- Mongo document classes are **separate** from domain classes. Map with `ToDocument()` / `ToDomain()` extensions. Never leak `BsonElement` attributes into Domain.
- One collection per aggregate root. Collection names are `camelCase` plural (`pipelines`, `jobRuns`).
- Indexes defined programmatically in a startup hook (`EnsureIndexesAsync`). Never rely on implicit indexes.
- Background workers implement `IHostedService` / `BackgroundService`. They resolve a fresh scope per tick — never hold long-lived scoped services.
- Each external dependency has its own subfolder (`Infrastructure/Mongo`, `Infrastructure/Keycloak`, `Infrastructure/Notifications`). No mixing.

## 7. Mocking policy

- Any external system we don't actually integrate with gets an interface in Application and a fake in `Infrastructure/Mocks/`. Naming: `IXxx` + `MockXxx`.
- Mocks return realistic-looking data (random, shaped correctly) and log what they would have done (`[mock notify] Would send email to ...`).
- Mocks are registered the same way as real impls — swapping is a one-line DI change.
- Every mock has a comment at the top explaining _what it would do in production_.

## 8. API layer rules

- Controllers inherit from `ControllerBase`. No Razor Pages, no Minimal APIs in this layered setup (Minimal APIs blur the controller/handler line).
- Routes are versioned: `/api/v1/...`. Bump the major version for breaking changes.
- Every action:
    - Has an `///` XML summary documenting what it does
    - Has XML `<param>` documentation for every parameter
    - Has XML `<response>` tags for each status code
    - Declares all responses via `[ProducesResponseType(typeof(T), StatusCodes.StatusXxx)]`
    - Declares `[Consumes("application/json")]` and `[Produces("application/json")]` at controller level
- Controllers are thin: they bind input, call the injected command/query handler, return the result. No business logic.
- Authorization uses `[Authorize(Roles = "...")]`. No inline role checks inside actions.
- Binding models (request DTOs) live in `Api/Contracts/Requests/`. They are distinct from Application DTOs — the wire format is owned by the API layer.

## 9. REST conventions

- **Resources are plural nouns:** `/pipelines`, `/alert-rules`. Verbs only for true actions that don't map to CRUD: `/pipelines/{id}/run`, `/pipelines/{id}/activate`.
- **HTTP methods:**
    - `GET` — read, safe, idempotent
    - `POST` — create a resource OR invoke a non-idempotent action
    - `PUT` — full replacement
    - `PATCH` — partial update
    - `DELETE` — delete, idempotent
- **Status codes:**
    - `200 OK` — successful read
    - `201 Created` — resource created; include `Location` header pointing to the new resource
    - `202 Accepted` — action queued for async processing
    - `204 No Content` — success with no body (e.g. DELETE)
    - `400 Bad Request` — validation / parse error
    - `401 Unauthorized` — no/invalid token
    - `403 Forbidden` — authenticated but not authorized
    - `404 Not Found` — resource doesn't exist
    - `409 Conflict` — unique constraint / state conflict
    - `422 Unprocessable Entity` — domain rule violation (state-machine, etc.)
- Error responses conform to RFC 7807 `ProblemDetails`. Include `traceId`.
- Pagination is **offset-based** unless a spec says otherwise: `?offset=0&limit=20`. Response shape: `{ items, total, offset, limit }`.
- List endpoints expose filters as query parameters, named in `camelCase`.

## 10. OpenAPI requirements

- Spec served at `/openapi/v1.json` in every environment.
- UI (Scalar or Swagger) served at `/scalar/v1` (or equivalent) in **Development only**.
- XML docs are fed into the generator via a document transformer so summaries/descriptions show up in the spec.
- Every `[ProducesResponseType]` must name an explicit type — no bare `StatusCodes.Status400BadRequest` without `typeof(ProblemDetails)`.
- The spec is exported to `docs/openapi.json` via a `--dump-openapi` CLI switch so consumers can generate types offline.

## 11. Validation & error handling

Two complementary layers of validation — use the lightest tool that can express the rule.

- **DataAnnotations on API request models** (`Api/Contracts/Requests/`). Use for simple, self-contained field constraints: `[Required]`, `[StringLength]`, `[Range]`, `[RegularExpression]`, `[EmailAddress]`. Enabled via ASP.NET's automatic model validation — no custom boilerplate needed, and the constraints flow straight into the OpenAPI schema. Prefer annotations over writing a FluentValidation rule for every trivial field check; they avoid verbose code.
- **FluentValidation in Application** for anything annotations can't express cleanly: cross-field rules, conditional validation, rules that need injected dependencies (e.g. "dataset must exist"), multi-step validation. Declared alongside the command/query it validates.
- **Domain rule violations**: typed exceptions from aggregates — never validation concerns leaking into Domain.
- A single `ExceptionHandlingMiddleware` in Api maps exceptions (including `ValidationException` from both layers) to `ProblemDetails` responses. Handlers and controllers never build `ProblemDetails` manually.
- Logs capture the exception, the request path, and the `traceId`. Response body shows a sanitized message (never the stack trace).

## 12. Configuration

- All settings live in `appsettings.json` and overrides in environment variables (`Mongo__ConnectionString`).
- Strongly-typed settings classes with `IOptions<T>`. Validated at startup (`services.AddOptions<T>().ValidateDataAnnotations().ValidateOnStart()`).
- Secrets never committed. Use User Secrets locally, environment variables or a vault in deployed environments.

## 13. Testing

- **Domain tests** are pure unit tests — no frameworks, no I/O. They cover every aggregate rule and every value-object invariant.
- **Application tests** fake repositories and external services with NSubstitute. They test the use-case orchestration, not the aggregate rules again.
- **Integration tests** use `WebApplicationFactory<Program>` + Testcontainers (real Mongo). They cover happy paths and the most important error cases for each endpoint.
- Coverage target: ≥ 70% on Domain and Application, ≥ 50% on Api integration tests.
- One test file per production class. Test names follow `Method_Scenario_Expectation` (`Transition_FromPendingToSuccess_Throws`).

## 14. Documentation

- Every public type, method, and property in Domain, Application, and Api carries XML doc comments. Internal helpers document the _why_.
- No pointless comments (`// set name` on `x.Name = ...`). Comment the hidden constraint, the workaround, the gotcha — not the obvious.
- Every project has a `README.md` covering how to run it, how it's structured, and the main extension points.

## 15. Naming

- Files, classes, methods: `PascalCase`.
- Interfaces: `IXxx`.
- Private fields: `_camelCase`.
- Constants: `PascalCase`.
- Async methods: suffix `Async`. Never use `.Result` or `.Wait()` — always `await`.
- One public type per file. Filename matches the type.
