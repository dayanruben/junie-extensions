---
name: "mongodb-patterns"
description: "MongoDB document design, Spring Data MongoDB, aggregations, indexing. Use when working with MongoDB."
---

# MongoDB Patterns Skill

Design efficient MongoDB documents and queries with Spring Data MongoDB.

## Scope and Boundaries

- Use this skill for document design, Spring Data MongoDB repositories, aggregations, indexing, and transactions.
- Use `spring-boot-patterns` for general service and configuration patterns.
- Use `spring-testing` for testing MongoDB repositories with Testcontainers.
- Examples use Spring Data MongoDB with Kotlin; adapt to Java as needed.

## When to Use
- Designing MongoDB document schemas
- Writing queries or aggregations
- Reviewing MongoDB repository code

---

## Document Design

```kotlin
@Document(collection = "users")
data class User(
    @Id val id: ObjectId = ObjectId(),
    val email: String,
    val name: String,
    @Indexed val createdAt: Instant = Instant.now(),
    val profile: UserProfile? = null,
)

data class UserProfile(
    val bio: String?,
    val avatarUrl: String?,
)
```

---

## Repository

```kotlin
interface UserRepository : MongoRepository<User, ObjectId> {
    fun findByEmail(email: String): User?
    fun findByCreatedAtAfter(date: Instant): List<User>

    @Query("{ 'profile.bio': { \$regex: ?0, \$options: 'i' } }")
    fun searchByBio(keyword: String): List<User>
}
```

---

## Indexing

```kotlin
@Document
@CompoundIndex(def = "{'status': 1, 'createdAt': -1}")
data class Order(
    @Id val id: ObjectId = ObjectId(),
    @Indexed val userId: ObjectId,
    val status: OrderStatus,
    val createdAt: Instant = Instant.now(),
)
```

---

## Aggregation

```kotlin
@Service
class OrderStatsService(private val mongoTemplate: MongoTemplate) {

    fun getOrderCountByStatus(): Map<String, Long> {
        val aggregation = Aggregation.newAggregation(
            Aggregation.group("status").count().`as`("count"),
            Aggregation.project("count").and("_id").`as`("status")
        )
        return mongoTemplate
            .aggregate(aggregation, "orders", Document::class.java)
            .mappedResults
            .associate { it.getString("status") to it.getLong("count") }
    }
}
```

---

## Indexing Strategy and explain()

Add indexes only when you have evidence of a slow query. Use `explain()` to verify index usage before and after.

```kotlin
// Check query plan via MongoDB shell or Compass:
// db.orders.find({ status: "PENDING" }).explain("executionStats")
// Look for: "winningPlan.stage" == "IXSCAN" (index used) vs "COLLSCAN" (full scan)

// ✅ Via MongoTemplate in tests/tooling
val query = Query.query(Criteria.where("status").`is`("PENDING"))
val explainResult = mongoTemplate.executeCommand(
    Document("explain", Document("find", "orders")
        .append("filter", Document("status", "PENDING")))
        .append("verbosity", "executionStats")
)
// Check explainResult["executionStats"]["totalDocsExamined"] vs ["totalDocsReturned"]
// If totalDocsExamined >> totalDocsReturned — add an index
```

When **not** to add an index:
- Low-cardinality fields (boolean, status with 2–3 values) on large collections where most rows match.
- Fields only used in rarely-executed admin queries.
- Collections with heavy write load where index maintenance cost outweighs read benefit.

---

## Transactions

`@Transactional` requires a **replica set** — a standalone MongoDB instance does not support multi-document transactions.

```kotlin
@Service
class TransferService(private val mongoTemplate: MongoTemplate) {

    @Transactional  // requires replica set
    fun transfer(fromId: ObjectId, toId: ObjectId, amount: BigDecimal) {
        mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").`is`(fromId)),
            Update().inc("balance", -amount),
            Account::class.java
        )
        mongoTemplate.updateFirst(
            Query.query(Criteria.where("_id").`is`(toId)),
            Update().inc("balance", amount),
            Account::class.java
        )
    }
}
```

**Dev environment setup** — run a single-node replica set locally with Docker:

```bash
docker run -d --name mongo-rs \
  -p 27017:27017 \
  mongo:7 --replSet rs0 --bind_ip_all

# Initialize the replica set (run once)
docker exec mongo-rs mongosh --eval "rs.initiate()"
```

```yaml
# application.yml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb?replicaSet=rs0
```
