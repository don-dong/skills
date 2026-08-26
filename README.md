# RuoYi Business Module Design

An AI coding skill for designing, extending, refactoring, and reviewing Java business modules in RuoYi and Spring Boot services.

It defines practical package layering, file naming, HTTP/service boundaries, MyBatis/MyBatis-Plus persistence rules, and safe model conversion practices.

## Install

```bash
npx skills add don-dong/ruoyi-design
```

Install only this skill:

```bash
npx skills add https://github.com/don-dong/skills --skill ruoyi-design
```

## Scope

Use this skill for module-specific Java code under `src/main/java`, including:

- Controllers and HTTP/RPC contracts;
- Service interfaces and implementations;
- MyBatis and MyBatis-Plus Mappers and Mapper XML;
- Request, Response, DTO, VO, Entity, Projection, runtime Model, Event, and Enum types;
- Module-local converters, components, factories, providers, exceptions, and configuration.

It does not prescribe frontend structure or replace existing shared-infrastructure ownership decisions.

Existing repository conventions always take precedence.

## Standard Layer Flow

The default persistence path is:

```text
Controller -> Service -> Mapper
```

A Repository is optional, not mandatory.

Use a Repository only when it provides meaningful persistence composition, such as:

- Coordinating multiple Mappers;
- Database and cache composition;
- Multiple data sources;
- External storage plus database persistence;
- Persistence-specific locking, retries, or cache invalidation.

Do not add a Repository that only forwards one Mapper call and invokes a Converter.

## HTTP and Service Boundary

When HTTP contracts and service contracts differ, use:

```text
Controller
  Request -> Converter -> DTO
  Service DTO -> VO
  Converter -> Response
```

Example:

```text
ChatCompletionRequest
  -> ChatCompletionDTO
  -> ChatCompletionVO
  -> ChatCompletionResponse
```

| Type | Primary responsibility |
|---|---|
| `Request` | HTTP/RPC input, validation, deserialization |
| `Response` | HTTP/RPC output, serialization, redaction |
| `DTO` | Service-layer or cross-module transfer type |
| `VO` | Service-layer output, detail, list, or presentation value |
| `Entity` | One physical database table mapping |
| `Projection` | Join, aggregate, computed, or read-only Mapper result |
| `Model` | Runtime execution/domain state |
| `Enum` | Finite module-owned state/type |

## Persistence Rules

### Single-table CRUD

Use an Entity and a MyBatis-Plus `BaseMapper<Entity>`:

```text
Entity + BaseMapper<Entity>
```

Example:

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

```java
@Mapper
public interface LLMModelMapper extends BaseMapper<LLMModel> {
}
```

Use this pattern for management CRUD of one table.

### Complex Join and Aggregate Queries

Do not force multi-table query results into a single-table Entity.

Use:

```text
Projection + Dedicated Mapper + Mapper XML
```

Example:

```text
llm_model
+ llm_provider
+ llm_model_route
+ llm_credential
= RoutableModelProjection
```

```java
@Mapper
public interface LLMCatalogMapper {

    List<RoutableModelProjection> selectRoutableModelByModelId(
            @Param("modelId") String modelId
    );

    List<AvailableModelProjection> selectAvailableModelList();
}
```

A Projection Mapper:

- Does not need a matching Entity;
- Does not need to extend `BaseMapper`;
- Uses Mapper XML for complex Join SQL;
- Returns a precise `XxxProjection` type.

## Naming Rules

- Packages use lowercase English:

```text
com.sten.llm
```

- Public types use `PascalCase`.
- Fields, methods, parameters, and local variables use `camelCase`.
- A Java file name must exactly match its public top-level type name.
- Service interface names use `XxxService`.
- Service implementation names use `XxxServiceImpl`.
- Mapper names use `XxxMapper`.
- Mapper XML namespace, statement IDs, parameter names, and return types must match the Mapper interface.
- Use correctly spelled packages:

```text
components
enums
```

Do not create:

```text
compoents
emus
```

### Acronym Convention

Follow the target module convention consistently.

For a module using uppercase `LLM`:

```text
package: com.sten.llm

LLMService
LLMServiceImpl
LLMChatController
LLMAdminController
LLMCatalogMapper
LLMException
ResolvedLLMInvocation
```

Do not mix:

```text
LLMService
LlmService
LlmCatalogMapper
```

inside one module.

## MyBatis-Plus and Lombok

When using MyBatis-Plus and Lombok:

- Use `@TableName`, `@TableId`, and `@TableField` for explicit table/field mappings;
- Use `autoResultMap = true` and a TypeHandler for JSON/JSONB Entity fields;
- Use `@Schema` for Entity and field descriptions when OpenAPI is available;
- Use `@Getter` and `@Setter` for Entity accessors;
- Avoid `@Data` on Entity types containing endpoint, token, credential, secret, or secret-reference fields;
- Prefer:

```java
@ToString(onlyExplicitlyIncluded = true)
```

- Only include safe fields in `toString()`.

Do not output these fields in normal logs or user responses:

```text
apiKey
secret
secretReference
token
credential
endpoint
endpointOverride
providerModel
defaultBaseUrl
```

A module that uses Lombok annotations must directly declare Lombok as a compile-time dependency. Do not rely on accidental transitive dependencies.

## Included Guidance

The complete skill provides:

- Package layout and model-placement rules;
- Controller, Service, Mapper, Repository, Converter, Component, Factory, Provider, Exception, Config, and Tool responsibilities;
- Request/Response, DTO/VO, Entity, Projection, and runtime Model boundaries;
- MyBatis-Plus Entity and `BaseMapper<Entity>` conventions;
- Projection Mapper and Mapper XML rules for multi-table Join queries;
- Naming and acronym conventions;
- Enum persistence rules;
- Prohibited patterns;
- Implementation workflow and pre-commit checklist.

See [SKILL.md](skills/ruoyi-design/SKILL.md) for the complete instructions loaded by supported AI coding agents.
