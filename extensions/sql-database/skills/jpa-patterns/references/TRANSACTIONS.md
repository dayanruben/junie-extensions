# JPA Transactions Reference

## Propagation Types

```java
@Service
public class OrderService {

    // REQUIRED (default): join existing or create new
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentService.processPayment(order); // joins this transaction
    }
}

@Service
public class PaymentService {

    // REQUIRES_NEW: always new transaction (independent commit/rollback)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(Order order) { ... }

    // MANDATORY: must run inside existing transaction (throws if none)
    @Transactional(propagation = Propagation.MANDATORY)
    public void auditPayment(Long orderId) { ... }

    // NOT_SUPPORTED: suspend current transaction, run without one
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void sendEmail(String to) { ... }
}
```

---

## Read-Only Transactions

```java
@Transactional(readOnly = true)
public List<OrderResponse> findAll() {
    // Benefits:
    // - Hibernate skips dirty checking
    // - DB driver may optimize (e.g. no write locks)
    // - Flush mode set to MANUAL
    return orderRepository.findAll().stream()
        .map(mapper::toResponse)
        .toList();
}
```

---

## Rollback Rules

```java
// Default: rolls back on RuntimeException and Error only
@Transactional
public void process() throws PaymentException { ... }  // WON'T rollback on PaymentException!

// Explicit rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void process() throws PaymentException { ... }  // rolls back

// Prevent rollback for specific exception
@Transactional(noRollbackFor = OptimisticLockException.class)
public void update() { ... }
```

---

## Common Mistakes

### Self-invocation bypass

```java
// ❌ BAD: calling @Transactional from same class bypasses Spring proxy
@Service
public class OrderService {
    public void processOrder(Long id) {
        updateOrder(id);  // @Transactional is IGNORED
    }

    @Transactional
    public void updateOrder(Long id) { ... }
}

// ✅ Fix: move transactional method to a separate service (preferred)
// Self-injection (@Autowired private OrderService self) technically works
// but creates a circular dependency — avoid it

// OrderService stays non-transactional for orchestration:
// public void processOrder(Long id) { orderUpdateService.updateOrder(id); }

// Separate service owns the transaction:
@Service
public class OrderUpdateService {
    @Transactional
    public void updateOrder(Long id) { ... }
}
```

### Transaction on private method

```java
// ❌ @Transactional on private method — silently ignored
@Transactional
private void saveInternal() { ... }

// ✅ Must be public (or at minimum package-private with class proxying)
@Transactional
public void saveInternal() { ... }
```

### Exception swallowing rolls back silently

```java
// ❌ BAD: exception is caught and not re-thrown — Spring thinks all is fine
// but data may be in inconsistent state
@Transactional
public void save(Order order) {
    try {
        orderRepository.save(order);
        auditService.log(order);  // throws
    } catch (Exception e) {
        log.error("Audit failed", e);
        // transaction commits without audit!
    }
}

// ✅ Mark for rollback manually if you catch exceptions
@Transactional
public void save(Order order) {
    try {
        orderRepository.save(order);
        auditService.log(order);
    } catch (Exception e) {
        log.error("Audit failed", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

---

## Optimistic vs Pessimistic Locking

```java
// Optimistic — @Version, no DB locks, fails fast on conflict
@Entity
public class Order {
    @Version
    private Long version;  // auto-incremented on each update
}

// Handle conflict
@Transactional
public Order update(Long id, UpdateRequest req) {
    try {
        Order order = orderRepository.findById(id).orElseThrow();
        order.setStatus(req.status());
        return orderRepository.save(order);
    } catch (OptimisticLockException e) {
        throw new ConflictException("Order was modified concurrently, please retry");
    }
}

// Pessimistic — DB-level lock (use when conflicts are frequent)
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findByIdForUpdate(@Param("id") Long id);
```
