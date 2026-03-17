# Junie Skills

Curated skill packs for [Junie](https://www.jetbrains.com/junie/) — the AI coding agent by JetBrains.

Skills are structured knowledge files that give Junie domain expertise: best practices, code patterns, anti-patterns, and checklists for specific technologies and tasks. When a skill is activated, Junie follows its guidance to produce higher-quality code and reviews.

## Repository Structure

```
jvm-base/              # Skills for JVM (Java/Kotlin) projects
  spring-boot-patterns/
  jpa-patterns/
  kotlin-code-style/
  ...
```

Each skill is a directory containing:
- `SKILL.md` — main skill file with frontmatter (`name`, `description`), patterns, and examples
- `references/` — (optional) detailed reference files linked from the main skill

## Skill Packs

### `jvm-base`

21 skills covering the core JVM ecosystem:

| Category | Skills |
|----------|--------|
| **Spring Boot** | `spring-boot-patterns`, `spring-cloud-patterns`, `spring-actuator-patterns`, `spring-testing` |
| **Data** | `jpa-patterns`, `sql-patterns`, `mongodb-patterns`, `redis-patterns` |
| **Security** | `security-audit`, `validation-patterns` |
| **Architecture** | `architecture-review`, `api-design-patterns`, `performance-patterns`, `concurrency-patterns` |
| **Messaging** | `messaging-patterns` |
| **DevOps** | `docker-patterns`, `kubernetes-patterns`, `ci-cd-patterns` |
| **Language** | `kotlin-code-style` |
| **Testing** | `test-quality` |
| **Debugging** | `debugging-investigation-patterns` |

## Skill Format

Each `SKILL.md` follows this structure:

```markdown
---
name: "skill-name"
description: "One-line description used for skill selection."
---

# Skill Title

## Scope and Boundaries
- What this skill covers
- When to defer to other skills

## When to Use
- Trigger conditions

## Core Patterns
- Code examples with ✅ good / ❌ bad annotations

## Anti-Patterns
- Common mistakes to avoid
```

## How Skills Are Selected

Skills are activated automatically based on project analysis:
- **Language detection** — Java/Kotlin source files and build configuration
- **Dependency detection** — build.gradle.kts, pom.xml, and version catalogs are scanned for library dependencies
- **File detection** — presence of Dockerfile, k8s manifests, CI configs, etc.

For example, a Spring Boot project with JPA and Redis will automatically receive: `spring-boot-patterns`, `jpa-patterns`, `sql-patterns`, `redis-patterns`, `security-audit`, `spring-testing`, and the core JVM skills.

## Contributing

When adding or modifying skills:

1. Follow the skill format above — include Scope/Boundaries, When to Use, and Anti-Patterns sections
2. Use code examples with `✅`/`❌` annotations to show good vs bad patterns
3. Keep examples as pattern fragments, not copy-paste production code
4. Add cross-references to related skills using backtick-quoted skill names (e.g., "see `jpa-patterns`")
5. Avoid duplicating content — reference other skills instead
6. Test that skill detection rules in Junie correctly activate the skill for target projects

## License

Copyright JetBrains s.r.o. All rights reserved.
