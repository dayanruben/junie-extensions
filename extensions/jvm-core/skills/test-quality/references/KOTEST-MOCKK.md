# Kotest + MockK Reference (Kotlin)

## Dependencies

```kotlin
// build.gradle.kts
testImplementation("io.kotest:kotest-runner-junit5:5.9.1")
testImplementation("io.kotest:kotest-assertions-core:5.9.1")
testImplementation("io.kotest.extensions:kotest-extensions-spring:1.3.0")
testImplementation("io.mockk:mockk:1.13.12")
```

---

## Kotest Styles

```kotlin
// BehaviorSpec — BDD style (Given/When/Then)
class UserServiceTest : BehaviorSpec({

    val repository = mockk<UserRepository>()
    val service = UserService(repository)

    given("a valid user ID") {
        `when`("findById is called") {
            then("should return the user") {
                val user = User(1L, "Alice")
                every { repository.findById(1L) } returns Optional.of(user)

                val result = service.findById(1L)

                result.name shouldBe "Alice"
                verify(exactly = 1) { repository.findById(1L) }
            }
        }
        `when`("user does not exist") {
            then("should throw ResourceNotFoundException") {
                every { repository.findById(99L) } returns Optional.empty()

                shouldThrow<ResourceNotFoundException> { service.findById(99L) }
            }
        }
    }
})

// FunSpec — simple function-based (closest to JUnit style)
class OrderServiceTest : FunSpec({

    val repo = mockk<OrderRepository>()
    val service = OrderService(repo)

    test("should create order") {
        every { repo.save(any()) } answers { firstArg() }
        val result = service.create(CreateOrderRequest("item-1", 2))
        result.status shouldBe OrderStatus.PENDING
    }
})

// DescribeSpec — nested describe/it blocks
class PaymentServiceTest : DescribeSpec({
    describe("processPayment") {
        it("succeeds with valid card") { ... }
        it("throws on expired card") { ... }
    }
})
```

---

## Kotest Assertions

```kotlin
// Basic
user.name shouldBe "Alice"
user.age shouldBeGreaterThan 18
result shouldBeInstanceOf UserResponse::class
value.shouldBeNull()
value.shouldNotBeNull()

// Collections
users shouldHaveSize 3
users shouldContain user
users.shouldBeEmpty()
users shouldContainExactlyInAnyOrder listOf(user1, user2)
users.forAll { it.active shouldBe true }

// Strings
str shouldStartWith "prefix"
str shouldContain "sub"
str shouldMatch Regex("\\d{4}-\\d{2}")

// Exceptions
shouldThrow<IllegalArgumentException> {
    service.create(invalidRequest)
}.message shouldContain "required"

shouldNotThrowAny { service.findById(1L) }

// Soft assertions
assertSoftly(user) {
    name shouldBe "Alice"
    email shouldContain "@"
    age shouldBeGreaterThan 0
}

// Recursive comparison
actual shouldBeEqualToComparingFields expected
```

---

## MockK

```kotlin
// Basic mock
val repository = mockk<UserRepository>()
every { repository.findById(1L) } returns Optional.of(user)
every { repository.save(any()) } answers { firstArg() }
every { repository.deleteById(any()) } just Runs

// Verify
verify(exactly = 1) { repository.findById(1L) }
verify(atLeast = 1) { repository.save(match { it.name == "Alice" }) }
confirmVerified(repository)

// Spy on real object
val service = spyk(UserService(repository))
every { service.heavyComputation() } returns "mocked"

// Capture arguments
val slot = slot<User>()
every { repository.save(capture(slot)) } answers { slot.captured }
service.create(request)
slot.captured.name shouldBe "Alice"

// Coroutines
coEvery { repository.findByIdSuspend(1L) } returns user
coVerify { repository.findByIdSuspend(1L) }

// Relaxed mock (all functions return defaults, no stubbing needed)
val repo = mockk<UserRepository>(relaxed = true)
```

---

## Spring Boot + Kotest

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest(
    @Autowired val mockMvc: MockMvc,
    @Autowired val userRepository: UserRepository,
) : FunSpec({

    afterEach { userRepository.deleteAll() }

    test("POST /users creates user and returns 201") {
        mockMvc.post("/api/v1/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name":"Alice","email":"alice@example.com","age":30}"""
        }.andExpect {
            status { isCreated() }
            jsonPath("$.name") { value("Alice") }
        }
    }

    test("GET /users/{id} returns 404 when not found") {
        mockMvc.get("/api/v1/users/999")
            .andExpect { status { isNotFound() } }
    }
})
```

---

## Coroutine Tests

```kotlin
class OrderServiceTest : FunSpec({

    test("should process order asynchronously") {
        val service = OrderService(mockk(relaxed = true))

        // runTest controls virtual time — no real delays
        runTest {
            val result = service.processAsync(CreateOrderRequest("item-1"))
            result.status shouldBe OrderStatus.COMPLETED
        }
    }
})
```
