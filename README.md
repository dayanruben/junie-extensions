# Junie Skills

Curated skill extensions for [Junie](https://www.jetbrains.com/junie/) — the AI coding agent by JetBrains.

Extensions give Junie domain expertise: best practices, code patterns, anti-patterns, and checklists for specific technologies. Each extension can include skills (knowledge files) and MCP servers (tools).

## Extensions

| Extension | Skills | MCP | Description |
|-----------|--------|-----|-------------|
| `context7` | 1 | Context7, DeepWiki | Up-to-date library docs and repository knowledge |
| `java-engineer` | 1 | — | Java 21 language features, idioms, and coding standards |
| `kotlin-engineer` | 1 + 8 references | — | Kotlin language, coroutines, Ktor, KMP, Android Compose, architecture |
| `spring-boot-engineer` | 1 + 9 references | — | Spring Boot 3.x — web, data, security, cloud, testing, reactive, resilience |
| `sql-engineer` | 1 | DBHub | SQL, Flyway/Liquibase migrations, query optimization |
| `redis-engineer` | 1 | — | Redis data structures, caching patterns, best practices |

## Combining Extensions

Extensions are designed to work independently or together without conflicts:

| Use Case | Extensions |
|----------|-----------|
| Java + Spring Boot | `java-engineer` + `spring-boot-engineer` + `sql-engineer` |
| Kotlin + Spring Boot | `kotlin-engineer` + `spring-boot-engineer` + `sql-engineer` |
| Pure Kotlin (Ktor / Android / KMP) | `kotlin-engineer` |
| Pure Java | `java-engineer` |
| Any project with Redis | + `redis-engineer` |
| Any project (always useful) | + `context7` |

## Structure

Each extension lives in `extensions/<name>/` and may contain:

```
extensions/<name>/
  extension.json          # name + description
  skills/<skill-name>/
    SKILL.md              # main skill file
    references/           # optional detailed references loaded on-demand
  mcp/
    .mcp.json             # optional MCP server definitions
```

### Skill Format

```markdown
---
name: "skill-name"
description: "One-line description."
---

# Title

## Reference Guide         ← hub-and-spoke: table pointing to references/
## Key Patterns            ← quick-start code examples
## Constraints             ← MUST DO / MUST NOT DO
```

Reference files are loaded on-demand based on context — only the relevant ones are read for each task.

## License

Copyright JetBrains s.r.o. All rights reserved.
