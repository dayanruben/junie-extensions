---
name: "spring-testing"
description: "Spring Boot testing patterns: @WebMvcTest, @DataJpaTest, @WebFluxTest, MockMvc, WebTestClient, StepVerifier, Testcontainers, @WithMockUser, JWT testing, AssertJ, BDDMockito, context caching, and version-sensitive migration notes. Use when writing or reviewing tests in a Spring Boot project."
---

# Spring Testing Skill

## Version Notes

- Examples assume modern Spring Boot / Spring Framework test support, but annotation names and package locations can differ between Boot 3.x and 4.x.
- Treat Boot migration examples as version-sensitive reference snippets, not universal defaults.
- Prefer the narrowest test slice that still proves the behavior you need; escalate to `@SpringBootTest` when slice tests would miss the relevant integration.

## When to Use
- Writing tests for Spring Boot controllers, repositories, or security
- Questions about `@WebMvcTest`, `@DataJpaTest`, `@WebFluxTest`, `@SpringBootTest`
- AssertJ assertions, BDDMockito, `ArgumentCaptor`
- Testcontainers setup, `@ServiceConnection`
- JWT/OAuth2/CSRF in tests, `@WithMockUser`
- Context caching, slow test suite, `@DirtiesContext`
- Boot 3 → 4 test migration (`@MockitoBean`, new package names)

---

## Routing Table

| Testing... | Read |
|---|---|
| AssertJ idioms, BDDMockito, `ArgumentCaptor`, `@MockitoBean` vs `@MockitoSpyBean`, context caching, testing pyramid, Boot 4 migration | `references/fundamentals.md` |
| JPA repositories, `@DataJpaTest`, Testcontainers, `@Transactional` traps, lazy loading, N+1, Hibernate 6/7 migration | `references/jpa-testing.md` |
| REST controllers, `@WebMvcTest`, `MockMvc`, `jsonPath`, `@RestControllerAdvice`, multipart | `references/mvc-testing.md` |
| Authentication, `@WithMockUser`, JWT, OAuth2, CSRF, `@PreAuthorize` in tests | `references/security-testing.md` |
| Reactive controllers, `@WebFluxTest`, `WebTestClient`, `StepVerifier`, SSE, virtual time | `references/webflux-testing.md` |
| WebSocket, STOMP, `WebSocketStompClient`, `BlockingQueue` pattern | `references/websocket-testing.md` |

---

## Slice Annotation Quick Reference

| Annotation | Loads | Typical startup |
|---|---|---|
| Unit test (no Spring) | Single class | < 100ms |
| `@WebMvcTest` | MVC layer only | 1–3s |
| `@DataJpaTest` | JPA + repositories only | 2–5s |
| `@WebFluxTest` | Reactive layer only | 1–3s |
| `@SpringBootTest` | Full context | 5–30s |
| `@SpringBootTest(RANDOM_PORT)` | Full context + real server | 10–30s+ |

**Rule of thumb**: prefer the narrowest slice that still validates the behavior you need. Escalate to `@SpringBootTest` when testing cross-cutting behavior (security + service + persistence together) or when a slice would hide the integration you care about.

---

## Key Rules (Always Apply)

```java
// ✅ BDDMockito over classic Mockito
given(repo.findById(1L)).willReturn(Optional.of(user));   // prefer
when(repo.findById(1L)).thenReturn(Optional.of(user));    // avoid

// ✅ then(mock).should() over verify()
then(emailService).should().sendWelcome(any(User.class));

// ✅ BigDecimal — always isEqualByComparingTo
assertThat(price).isEqualByComparingTo(BigDecimal.valueOf(9.99));

// ✅ Version-sensitive example — verify against the current Boot version
@MockitoBean UserService userService;   // Boot 4 style
@MockBean UserService userService;      // Boot 3 style
```

**Context caching**: every unique combination of mock/spy bean overrides can create a new context. Standardize override sets across related test classes to share a context and speed up the suite.
