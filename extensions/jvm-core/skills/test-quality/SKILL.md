---
name: "test-quality"
description: "Write high-quality tests for non-Spring JVM projects: JUnit 5 + AssertJ for Java, Kotest + MockK for Kotlin. Use when user says \"add tests\", \"write tests\", \"improve test coverage\". For Spring Boot projects use spring-testing instead."
---

# Test Quality Skill

## When to Use
- Writing tests in a **non-Spring** JVM project
- Kotest / MockK patterns for Kotlin
- User asks to "add tests" / "improve test coverage"

> For Spring Boot projects (with `@WebMvcTest`, `@DataJpaTest`, etc.) use the `spring-testing` skill instead.

## Framework Preferences

| Language | Test Framework | Assertions | Mocking |
|---|---|---|---|
| Java | JUnit 5 (Jupiter) | AssertJ | Mockito |
| Kotlin | Kotest | Kotest assertions | MockK |

Full examples → [references/JUNIT5-ASSERTJ.md](references/JUNIT5-ASSERTJ.md) · [references/KOTEST-MOCKK.md](references/KOTEST-MOCKK.md)

---

## AAA Pattern

Prefer Arrange-Act-Assert for readability and consistency:

```java
@Test
@DisplayName("Should load plugin from valid directory")
void shouldLoadPluginFromValidDirectory() {
    // Arrange
    Path pluginDir = Paths.get("test-plugins/valid-plugin");
    PluginLoader loader = new DefaultPluginLoader();

    // Act
    Plugin plugin = loader.load(pluginDir);

    // Assert
    assertThat(plugin)
        .isNotNull()
        .extracting(Plugin::getId, Plugin::getVersion)
        .containsExactly("test-plugin", "1.0.0");
}
```

---

## Naming Conventions

```
// Class: <ClassUnderTest>Test
UserServiceTest

// Method options:
void should_throwException_when_pluginDirectoryNotFound() { }   // descriptive
void shouldLoadPlugin() + @DisplayName("...")                   // clean + readable
```

---

## Anti-patterns

```java
// ❌ Generic names
@Test void test1() { }

// ❌ Multiple unrelated assertions in one test
@Test void testEverything() { /* 50 unrelated assertions */ }

// ❌ Swallowing exceptions
catch (Exception e) { /* empty */ }

// ❌ Testing implementation details
assertThat(service.internalCache).hasSize(3);
```

---

## Coverage Guidelines

- **Target**: aim for strong coverage of core business logic (often 70-80%+ is a reasonable heuristic, not a hard rule)
- **Focus**: public APIs, error handling, edge cases, integration points
- **Skip**: trivial getters/setters, generated code, simple POJOs

---

## Best Practices

1. One concept per test
2. Descriptive names with `@DisplayName`
3. Extract helpers for common setup (`@BeforeEach`, factory methods)
4. Use `@Nested` for logical grouping
5. Parameterize similar tests
6. Test behaviour, not implementation
