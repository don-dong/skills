---
name: ruoyi-design
description: Design, extend, refactor, or review Java business-module boundaries in RuoYi/Spring Boot services, including package placement and HTTP, service, domain, and persistence models. Use for module-specific application code; not for shared infrastructure or frontend work.
---

# RuoYi Business Module Design

Use this skill to design or review the Java package structure of a RuoYi-style Spring Boot business module. Keep class responsibilities clear, make related code easy to locate, and prevent HTTP contracts or persistence details from leaking across layers.

This skill applies to module-specific code under `src/main/java`. Shared infrastructure, cross-module contracts, and capabilities reused across modules belong in the existing shared module or public API, not in a business module.

## When To Use

- Creating or extending a business module or backend capability.
- Adding a Controller, Service, Mapper, Request, Response, DTO, VO, runtime model, or entity.
- Deciding whether a class belongs in `domain`, `repository`, `factory`, `tools`, or a shared module.
- Refactoring an oversized module, removing cyclic dependencies, or reviewing package and model boundaries.

## Decision Rules

1. **Existing conventions win.** Inspect the target module and adjacent modules before adding packages, base classes, mapper XML, conversion code, or shared components. Do not introduce a second local convention without a concrete migration plan.
2. **Organize by business module first.** For a new module with no established project convention, keep its layers under one root package, normally `com.<project>.<module>`. Do not force this layout onto a project that already uses another consistent structure.
3. **Keep one primary responsibility per class.** Controllers adapt protocols, Services express business use cases, Mappers perform persistence access, and domain types express data or domain concepts.
4. **Model boundaries deliberately.** Do not expose entities as HTTP request or response types. Create separate Request/Response, DTO, VO, and runtime models only when their responsibilities or independent evolution justify them; do not duplicate identical models by default.
5. **Keep dependencies inward.** The normal flow is `controller -> service -> repository (optional) -> mapper`. Domain types can be used by higher layers, but must not depend on controllers or service implementations.
6. **Reuse based on ownership, not a numeric threshold.** Move a stable, genuinely shared capability to an existing shared module or public contract when it has clear cross-module ownership. Do not copy it into each module.

## Suggested Package Layout

Use this layout for a new module only when the repository has no conflicting convention. Create optional packages only when they solve a real module-specific need.

```text
src/main/java/com/<project>/<module>/
|- controller/                         # HTTP or RPC adapters
|- service/                            # business-use-case contracts
|  `- impl/                            # use-case implementations
|- mapper/                             # MyBatis mapper interfaces
|- domain/
|  |- query/
|  |  |- request/                      # HTTP or RPC input models
|  |  `- response/                     # HTTP or RPC output models
|  |- dto/                             # layer or cross-module transfer models
|  |- vo/                              # internal read or presentation models
|  |- model/                           # runtime domain models
|  |- entity/                          # persistence entities
|  `- enums/                           # module-owned enums
|- repository/                         # optional persistence composition
|- exception/                          # module-specific business exceptions
|- factory/                            # optional complex construction or strategy selection
|- config/                             # module-only Spring configuration
`- tools/                              # module-only, stateless utilities
```

### Naming

- Use lowercase English package names without hyphens or underscores.
- Use `PascalCase` for types and `camelCase` for fields and methods.
- Prefer a business noun plus a role suffix: `CreateUserRequest`, `UserResponse`, `CreateUserDTO`, `UserService`, and `UserMapper`.
- Name the service contract `XxxService` and its default implementation `XxxServiceImpl`. Avoid redundant names such as `IService` or `ServiceInterface`.
- Name mapper interfaces `XxxMapper`. Keep matching mapper XML in the repository's established resource location and keep its namespace, method names, parameters, and return types synchronized.
- Use `enums`, not a misspelling such as `emus`. For a legacy module, migrate deliberately rather than creating both packages.
- Prefer a specific utility name such as `UserNameNormalizer` over an ever-growing `Util` or `Helper` class.

## Layer Responsibilities

### Controller

- Define routes, HTTP methods, authorization annotations, parameter binding, and request validation.
- Accept a Request model or the repository's established boundary model, then convert it to the service input when necessary.
- Convert service output to a Response model when the API needs a distinct contract, such as field redaction, renaming, wrapping, or version compatibility.
- A stable VO may be returned directly only when the project convention permits it and it is intentionally the API contract. Do not also create an identical Response model.
- Do not implement business rules, call a Mapper or Repository directly, manage transactions, or assemble complex SQL.
- Reuse the project's existing pagination, authorization, data-scope, and response wrappers.

### Service And Service Implementation

- The service contract exposes business use cases such as `create`, `publish`, `disable`, and `findDetail`, rather than mirroring every mapper CRUD operation.
- Service inputs should be DTOs or explicit scalar identifiers. Outputs should be VOs, runtime models, or the project's standard page result. Request and Response types are normally controller-boundary types.
- The implementation coordinates validation, ownership checks, state transitions, transactions, persistence calls, and conversions.
- Annotate only implementation classes with `@Service`. Use constructor injection with `private final` dependencies; do not use field injection.
- Place `@Transactional` on the smallest service method that must be atomic.
- Do not expose HTTP objects, entities, MyBatis pages, or other infrastructure details to the controller layer.
- Call another module through its public API or contract, never through its Mapper, Entity, or service implementation.

### Mapper And Repository

- A Mapper provides module-local MyBatis access. Its methods use entities, dedicated query DTOs, explicit parameters, or read-only projections; it never depends on HTTP Request or Response models.
- Keep business validation, permission decisions, state transitions, and cross-table business orchestration outside Mappers.
- Do not use Mappers as cross-module APIs.
- Add a Repository only for meaningful persistence composition: multiple Mappers, aggregates, caching, multiple data sources, or external storage. Do not add a pass-through Repository for each single-table Mapper.
- A Repository may depend on Mappers, but must not contain HTTP or response-format concerns.

### Domain Models

- `domain.query.request`: HTTP or RPC input structures, validation, and deserialization rules.
- `domain.query.response`: HTTP or RPC output structures, serialization, redaction, and response-specific formatting.
- `domain.dto`: transfer data for service-layer or cross-module contracts. Use focused `CreateXxxDTO`, `UpdateXxxDTO`, and `XxxQueryDTO` types instead of a broad nullable "save" DTO.
- `domain.vo`: internal list, detail, option, or aggregate-query results. Expose only data needed by the relevant flow.
- `domain.model`: immutable, runtime business models such as execution contexts, aggregates, or normalized remote-call data. They are not HTTP contracts, table mappings, or stable cross-module APIs.
- `domain.entity`: database-table mappings for internal persistence use. Do not use an entity as an HTTP input or output type.
- `domain.enums`: finite module-owned states or types. Use singular enum names and `UPPER_SNAKE_CASE` constants. Persist explicit codes and conversions, never `ordinal()`. Move genuinely shared enums to a shared contract module.

### Factory, Exception, Config, And Tools

- Add a Factory for complex construction, invariant creation, normalization, defaults, or strategy selection. Do not create one for trivial field copies.
- Put module-recognizable business exceptions in `exception`. Include useful context without logging secrets, tokens, SQL, or personal data. Reuse the project's common validation, authorization, and framework exceptions.
- Put module-only Spring configuration and configuration properties in `config`. Put global MVC, security, database, Redis, and exception-handling configuration in existing shared infrastructure.
- Keep `tools` stateless and side-effect free. Tools must not use Spring injection, database or network I/O, transactions, or business workflows.

## Prohibited Dependencies And Patterns

- A Controller calling a Mapper or Repository directly.
- A Mapper calling a Service, Controller, or Factory, or implementing business rules.
- Inheriting Request, Response, Entity, DTO, VO, or runtime Model merely to reuse fields. Use explicit conversion, MapStruct, or the repository's existing conversion approach.
- Exposing Mapper, Entity, or MyBatis `Page` details through the service API to the controller layer.
- Using `tools` for I/O, Spring beans, or business orchestration.
- Duplicating common exceptions, global configuration, generic dictionaries, date or security utilities, or shared enums inside a business module.
- Accessing another module's Mapper, Entity, or service implementation through Java package imports.

## Implementation Workflow

1. Confirm module ownership and inspect existing adjacent and shared capabilities.
2. Define only the model boundaries the use case requires and name the service use case.
3. Extend the entity, mapper, and matching mapper XML as needed; check whether SQL changes require a database migration.
4. Implement validation, transaction boundaries, state changes, and conversions in the service implementation. Add a Factory or Repository only when it adds meaningful separation.
5. Add the controller route, authorization, request validation, and intentional boundary conversion.
6. Verify that entities do not leak, Mappers remain module-local, shared behavior is not duplicated, and no dependency points outward.
7. Compile the affected Maven module and add or update focused service, mapper/repository, and controller tests as appropriate.

## Pre-Commit Checklist

- [ ] The code follows the target repository's established module and package conventions.
- [ ] Each new class has one clear responsibility and a specific, conventional name.
- [ ] HTTP models, transfer models, read models, runtime models, and entities are separated only where their responsibilities differ.
- [ ] No API accepts or returns an entity, and request validation covers required boundary rules.
- [ ] Controllers do not access Mappers or Repositories and do not contain business orchestration.
- [ ] Service APIs express use cases; implementations use constructor injection and declare transactions where needed.
- [ ] Mapper XML, when used, matches its Mapper interface exactly.
- [ ] Optional Repository, Factory, Exception, Config, and Tools packages exist only for real module-specific needs.
- [ ] Cross-module collaboration uses public APIs or contracts, and shared capabilities are not duplicated.
- [ ] There are no reverse dependencies, cyclic dependencies, broad catch-all models, or cross-layer model leaks.

## Final Principle

Follow the repository's existing module boundaries and shared capabilities first. Within those constraints, code should make its business purpose, layer, and allowed dependencies clear from its package and type name.
