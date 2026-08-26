---
name: ruoyi-design
description: Design, extend, refactor, or review Java RuoYi/Spring Boot business modules. Use for package placement, file naming, HTTP/service/persistence boundaries, MyBatis-Plus entities, projections, and model conversion.
---

# RuoYi Business Module Design

Use this skill for module-specific Java code under `src/main/java`. Make class responsibility, dependency direction, persistence shape, and API exposure clear from package and type names.

Shared infrastructure, reusable cross-module capabilities, and public contracts belong in an existing common/API module, not by copying code into a business module.

## Core Rules

1. **Existing conventions win.** Inspect the target module, adjacent modules, POM dependencies, Mapper/XML conventions, and naming before adding code.
2. **One class, one primary responsibility.** Controllers adapt HTTP; services coordinate use cases; mappers read/write persistence; converters transform boundaries; domain types describe data.
3. **Dependencies point inward.** Default flow is `controller -> service -> mapper`. A repository is optional, not mandatory.
4. **Do not rely on accidental transitive dependencies.** A module directly importing a common module's type must directly declare that common module dependency.
5. **Do not leak internals.** Entity, Projection, runtime Model, endpoint, credential reference, provider model name, secret, and SDK type are not ordinary user API responses.
6. **Only add models for real boundaries.** Do not duplicate structurally identical Request, DTO, VO, Response, Entity, or Model classes without a distinct role.

## Package Layout

Create optional packages only for actual module needs.

```text
src/main/java/com/<project>/<module>/
├── controller/                         # HTTP/RPC adapters
├── service/                            # use-case contracts
│   └── impl/                           # service implementations
├── mapper/                             # MyBatis/MyBatis-Plus interfaces
├── converter/                          # stateless boundary conversion
├── components/                         # focused Spring business components
├── factory/                            # complex construction/strategy selection
├── provider/                           # external provider abstractions
├── domain/
│   ├── query/
│   │   ├── request/                    # HTTP/RPC input
│   │   └── response/                   # HTTP/RPC output
│   ├── dto/                            # service/cross-module transfer types
│   ├── vo/                             # service-layer output/read values
│   ├── entity/                         # single-table persistence entities
│   ├── projection/                     # Join/aggregate SQL result types
│   ├── model/                          # immutable runtime/execution models
│   ├── event/                          # module stream/event payloads
│   └── enums/                          # finite module-owned states/types
├── exception/                          # module business exceptions
├── handler/                            # HTTP exception handlers
├── config/                             # module Spring configuration
└── utils/                              # dependency-free helpers only
```

## Model Placement

| Model | Package | Responsibility |
|---|---|---|
| `XxxRequest` | `domain.query.request` | HTTP/RPC binding, validation, deserialization |
| `XxxResponse` | `domain.query.response` | HTTP/RPC serialization and redaction |
| `XxxDTO` | `domain.dto` | Service/cross-module transfer input/output |
| `XxxVO` | `domain.vo` | Service-layer output/list/detail value |
| `Xxx` for one table | `domain.entity` | One physical table persistence mapping |
| `XxxProjection` | `domain.projection` | Join, aggregate, computed, read-only Mapper result |
| `XxxModel`, `ResolvedXxx` | `domain.model` | Runtime execution/domain state |
| `XxxStatus`, `XxxType` | `domain.enums` | Explicit finite state/type |

## Naming Rules

- Packages are lowercase English without hyphens or underscores.
- Public types use `PascalCase`; fields, methods, parameters, and local variables use `camelCase`.
- A public top-level type's filename must exactly match its type name.
- Services use `XxxService` and `XxxServiceImpl`; do not use `IService` or `ServiceInterface`.
- Mapper interfaces use `XxxMapper`; matching XML namespace, statement ID, parameter names, and return types must agree exactly.
- Use the correctly spelled `components` and `enums` packages. Never create `compoents` or `emus`.

### Acronyms

Follow the target module's acronym convention consistently. If the module uses uppercase `LLM`, package names remain lowercase but all matching type names preserve uppercase `LLM`.

```text
package: com.sten.llm
class:   LLMService
class:   LLMServiceImpl
class:   LLMChatController
class:   LLMCatalogMapper
class:   LLMException
class:   ResolvedLLMInvocation
```

Do not mix `LLMService`, `LlmService`, and `LlmCatalogMapper` in one module.

## HTTP and Service Boundaries

When HTTP and service contracts differ, use:

```text
Controller
  Request -> Converter -> DTO
  Service DTO -> VO
  Converter -> Response
```

Example:

```text
ChatCompletionRequest -> ChatCompletionDTO
ChatCompletionDTO -> ChatCompletionVO
ChatCompletionVO -> ChatCompletionResponse
```

### Controller

Controllers define routes, authorization, request validation, request/response conversion, response wrappers, and SSE/HTTP encoding.

Controllers must not call Mapper or Repository directly, implement business rules, manage transactions, or build SQL. Do not return Entity, Projection, endpoint, credential reference, secret, provider SDK type, or stack trace.

### Service

Services expose named use cases, receive DTOs or scalar identifiers, and return VOs/runtime values. They coordinate business validation, Mapper calls, access policy, transactions, provider calls, and runtime model assembly.

- Annotate only implementations with `@Service`.
- Use constructor injection and `private final` dependencies.
- Put `@Transactional` on the smallest atomic implementation method.
- Do not receive HTTP Request or return HTTP Response after a DTO/VO boundary is established.

### Converter

`converter/` contains module-local stateless transformations:

```text
Request -> DTO
VO -> Response
Projection -> Runtime Model
```

Converters must not use Spring injection, database/network I/O, transactions, or business workflows.

## Persistence Rules

### Entity and Single-table Mapper

An Entity maps exactly one table and is for persistence/management CRUD, not a user HTTP response.

For MyBatis-Plus plus Lombok:

```java
@Getter
@Setter
@ToString(onlyExplicitlyIncluded = true)
@TableName("llm_model")
@Schema(description = "LLM")
public class LLMModel {

    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    @TableField("model_id")
    private String modelId;
}
```

Rules:

- Use `@TableName`, `@TableId`, and `@TableField` for explicit mappings.
- Use `autoResultMap = true` and a project TypeHandler for JSON/JSONB Entity fields.
- Use `@Schema` descriptions when OpenAPI is available.
- Avoid Lombok `@Data` for Entity types containing endpoint, credential, token, secret, or secret-reference fields.
- Use `@ToString(onlyExplicitlyIncluded = true)` and include only safe fields.
- A module using Lombok annotations declares Lombok directly as a compile-time dependency.

A single-table Mapper should extend `BaseMapper<Entity>`:

```java
@Mapper
public interface LLMModelMapper extends BaseMapper<LLMModel> {
}
```

Use table mappers for management CRUD:

```text
LLMProviderMapper    -> LLMProvider / llm_provider
LLMModelMapper       -> LLMModel / llm_model
LLMCredentialMapper  -> LLMCredential / llm_credential
LLMModelRouteMapper  -> LLMModelRoute / llm_model_route
```

### Projection and Complex Query Mapper

Do not force a multi-table query into a single-table Entity.

```text
llm_model + llm_provider + llm_model_route + llm_credential
= RoutableModelProjection
```

Use `domain.projection` for Join, aggregate, computed, and read-only SQL results. A projection Mapper has no matching Entity and does not extend `BaseMapper` merely because related Entity classes exist.

```java
@Mapper
public interface LLMCatalogMapper {
    List<RoutableModelProjection> selectRoutableModelByModelId(
            @Param("modelId") String modelId);

    List<AvailableModelProjection> selectAvailableModelList();
}
```

Place complex SQL in matching XML under `resources/mapper/**`. Keep joins, filters, sorting, and result maps in XML; keep permission checks, validation, and route selection outside Mapper.

### Mapper Method Names

Use operation prefixes and business conditions:

```text
selectLLMModelById
selectLLMModelByModelId
selectLLMModelList
selectRoutableModelByModelId
selectAvailableModelList
insertLLMModel
updateLLMModel
deleteLLMModelById
```

Do not redeclare inherited MyBatis-Plus methods such as `selectById`, `insert`, `updateById`, or `deleteById`.

### Repository Is Optional

Default flow:

```text
controller -> service -> mapper
```

Add `repository/` only for real persistence composition: multiple Mappers, cache/database coordination, multiple data sources, external storage coordination, persistence-specific locking, or cache invalidation.

Do not add a pass-through Repository that only invokes one Mapper and a converter. Inject the Mapper into the Service directly.

## Components, Factory, Provider, Exception, and utils

- `components/`: focused Spring collaborators such as validators, access policies, and route selectors.
- `factory/`: complex construction/defaults/strategy selection, not SQL or access policy.
- `provider/`: module-owned external capabilities, such as credential resolution.
- `exception/`: module business exceptions; `handler/`: safe HTTP error translation.
- `utils/`: dependency-free stateless helpers only.

Credentials, endpoints, secret references, provider raw errors, and stack traces must not be exposed to normal user responses or ordinary logs.

## Enums

Use `domain.enums` for finite module-owned database states:

```text
ProviderProtocol
CredentialStorageType
CredentialStatus
RouteHealthStatus
```

Use singular enum names and `UPPER_SNAKE_CASE` constants. Persist explicit names/codes; never persist `ordinal()`. Keep Entity state fields as `String` until an enum TypeHandler is intentionally implemented.

## Prohibited Patterns

- Controller calling Mapper or Repository directly.
- Mapper calling Controller, Service, Component, Factory, or Provider.
- Service accepting Request or returning Response after DTO/VO boundaries exist.
- Returning Entity, Projection, endpoint, secret reference, credential, provider model, or SDK type to a user API.
- Treating a Join Projection as an Entity.
- Forcing a Projection Mapper to extend `BaseMapper`.
- Adding a pass-through Repository.
- Putting Spring beans, I/O, transactions, or business workflows in `converter` or `utils`.
- Mixing acronym naming conventions in one module.

## Workflow

1. Inspect module conventions, Maven dependencies, existing entities/mappers, and adjacent modules.
2. Define the use case and required Request/Response, DTO/VO, Entity, Projection, and runtime Model boundaries.
3. Use Service -> Mapper by default; add Repository only for real composition.
4. Add Flyway migration before table access code.
5. Add Entity plus `BaseMapper<Entity>` for each table that needs management CRUD.
6. Add Projection plus XML Mapper for complex Join/aggregate runtime reads.
7. Add stateless converters and focused Components.
8. Implement `XxxService` / `XxxServiceImpl` using DTO input and VO output where applicable.
9. Implement Controller conversion, validation, authorization, and protocol encoding.
10. Compile affected Maven modules and add focused Converter, Mapper, Service, and Controller tests.

## Pre-Commit Checklist

- [ ] Package/type names and acronym casing follow the target module convention.
- [ ] Public filenames match public type names.
- [ ] Controller uses Request/Response; Service uses DTO/VO where the boundary exists.
- [ ] Entity maps one table and is never returned to a user API.
- [ ] Join/aggregate SQL returns `XxxProjection`.
- [ ] Single-table MyBatis-Plus Mappers extend `BaseMapper<Entity>`.
- [ ] Projection Mapper XML namespace, IDs, parameters, result maps, and types match its interface.
- [ ] Repository exists only when it provides meaningful persistence composition.
- [ ] Converter is stateless and has no Spring/I/O/transaction behavior.
- [ ] `@ToString` excludes endpoints, secrets, secret references, tokens, and credentials.
- [ ] Direct Maven dependencies exist for directly imported common APIs and annotation processors.
- [ ] Affected modules compile and focused tests pass.
