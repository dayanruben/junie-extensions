# JUnit 5 + AssertJ + Mockito Reference

> **Scope**: This reference covers AssertJ and Mockito for **non-Spring** unit tests (plain JUnit 5 + `@ExtendWith(MockitoExtension.class)`).
> For Spring Boot tests (`@WebMvcTest`, `@DataJpaTest`, `@MockitoBean`, BDDMockito, context caching), see `spring-testing` skill instead.

## AssertJ Quick Reference

```java
// Basic
assertThat(value).isEqualTo(expected);
assertThat(value).isNotNull();
assertThat(number).isPositive().isGreaterThan(5);

// Collections
assertThat(list).hasSize(3).contains(item).doesNotContain(other);
assertThat(list).containsExactly(a, b, c);
assertThat(list).containsExactlyInAnyOrder(c, a, b);
assertThat(list).allMatch(predicate);
assertThat(list).filteredOn(p -> p.isActive()).extracting(User::getName).contains("Alice");

// Strings
assertThat(str).isNotBlank().startsWith("prefix").contains("sub").matches("regex\\d+");

// Exceptions
assertThatThrownBy(() -> service.call())
    .isInstanceOf(NotFoundException.class)
    .hasMessageContaining("not found");
assertThatNoException().isThrownBy(() -> service.call());

// Objects — field comparison
assertThat(actual).usingRecursiveComparison().ignoringFields("id", "createdAt").isEqualTo(expected);

// Multi-property extraction
assertThat(plugin)
    .extracting(Plugin::getId, Plugin::getVersion, Plugin::getState)
    .containsExactly("my-plugin", "1.0", PluginState.STARTED);

// Soft assertions (all evaluated even if some fail)
SoftAssertions softly = new SoftAssertions();
softly.assertThat(user.getName()).as("name").isNotBlank();
softly.assertThat(user.getEmail()).as("email").contains("@");
softly.assertAll();
```

---

## Mockito

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository repository;

    @InjectMocks
    private UserServiceImpl service;

    @Test
    void shouldReturnUserById() {
        // Given
        User user = new User(1L, "Alice");
        when(repository.findById(1L)).thenReturn(Optional.of(user));

        // When
        UserResponse result = service.findById(1L);

        // Then
        assertThat(result.getName()).isEqualTo("Alice");
        verify(repository).findById(1L);
    }

    @Test
    void shouldThrowWhenNotFound() {
        when(repository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.findById(99L))
            .isInstanceOf(ResourceNotFoundException.class);
    }
}
```

---

## Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"1.0.0", "2.1.3", "10.0.0-SNAPSHOT"})
void shouldAcceptValidVersions(String version) {
    assertThat(VersionParser.isValid(version)).isTrue();
}

@ParameterizedTest
@CsvSource({
    "alice@example.com, true",
    "not-an-email,      false",
    ",                  false"
})
void shouldValidateEmail(String email, boolean expected) {
    assertThat(validator.isValid(email)).isEqualTo(expected);
}

@ParameterizedTest
@MethodSource("invalidRequests")
void shouldRejectInvalid(CreateUserRequest req, String expectedError) {
    assertThatThrownBy(() -> service.create(req))
        .hasMessageContaining(expectedError);
}

static Stream<Arguments> invalidRequests() {
    return Stream.of(
        Arguments.of(requestWithBlankName(), "Name is required"),
        Arguments.of(requestWithInvalidEmail(), "Invalid email")
    );
}
```

---

## Nested Tests

```java
@DisplayName("UserService")
class UserServiceTest {

    @Nested
    @DisplayName("when creating a user")
    class WhenCreating {

        @Test
        @DisplayName("should return created user")
        void shouldReturnCreated() { ... }

        @Test
        @DisplayName("should throw when email already exists")
        void shouldThrowOnDuplicate() { ... }
    }

    @Nested
    @DisplayName("when deleting a user")
    class WhenDeleting {

        @Test
        @DisplayName("should throw when user not found")
        void shouldThrowWhenNotFound() { ... }
    }
}
```

---

## Async Tests

```java
@Test
void shouldCompleteAsync() {
    CompletableFuture<String> future = service.processAsync(request);

    assertThat(future)
        .succeedsWithin(Duration.ofSeconds(5))
        .satisfies(result -> assertThat(result).isNotBlank());
}
```

---

## JaCoCo Coverage (Maven)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution><goals><goal>prepare-agent</goal></goals></execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```
