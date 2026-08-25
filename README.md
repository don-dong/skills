# RuoYi Business Module Design

An agent skill for designing, extending, refactoring, and reviewing Java business-module boundaries in RuoYi and Spring Boot services.

It helps agents place code in appropriate packages, preserve layer boundaries, and keep HTTP contracts, service use cases, domain models, and persistence entities separate.

## Install

```bash
npx skills add don-dong/ruoyi-design
```

To install it for a specific agent:

```bash
npx skills add don-dong/ruoyi-design --skill ruoyi-design --agent codex
```

## Use Cases

- Create or extend a RuoYi/Spring Boot business module.
- Add controllers, services, MyBatis mappers, DTOs, VOs, request/response models, entities, or enums.
- Review package placement, dependency direction, and model leakage.
- Refactor large modules, duplicated utilities, or cross-module dependencies.

## Scope

The skill applies to module-specific Java application code under `src/main/java`. It does not prescribe shared infrastructure, frontend architecture, or database-specific optimization.

Existing repository conventions take precedence. For a new module without an established convention, the skill provides a suggested package layout and layer responsibilities.

## Included Guidance

- Controller, service, mapper, repository, factory, configuration, exception, and utility responsibilities.
- Boundaries between Request/Response, DTO, VO, runtime model, and Entity types.
- Dependency and cross-module access rules.
- An implementation workflow and pre-commit checklist.

See [SKILL.md](SKILL.md) for the complete instructions loaded by supported AI coding agents.
