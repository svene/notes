# Quarkus CDI Events vs Spring ApplicationEventPublisher

## Overview

Both Spring Boot and Quarkus support an event-driven programming model to decouple components. Spring uses `ApplicationEventPublisher`; Quarkus uses **CDI Events** from the Jakarta EE/CDI specification — making it more standards-based.

---

## Publishing Events

### Spring
```java
@Autowired
private ApplicationEventPublisher publisher;

publisher.publishEvent(new OrderPlacedEvent(order));
```

### Quarkus
```java
@Inject
Event<OrderPlacedEvent> orderEvent;

orderEvent.fire(new OrderPlacedEvent(order));          // synchronous
orderEvent.fireAsync(new OrderPlacedEvent(order));     // asynchronous
```

---

## Receiving Events

### Spring
```java
@EventListener
public void onOrderPlaced(OrderPlacedEvent event) { ... }
```

### Quarkus
```java
public void onOrderPlaced(@Observes OrderPlacedEvent event) { ... }

// Async version
public CompletionStage<Void> onOrderPlacedAsync(@ObservesAsync OrderPlacedEvent event) { ... }
```

---

## Quarkus-Specific Features

### Qualifiers (event filtering)
CDI qualifiers allow observers to subscribe only to specific subtypes of events — something Spring doesn't support natively.

```java
@Inject @Cancelled Event<OrderEvent> cancelledEvent;
cancelledEvent.fire(new OrderEvent(order));

public void onCancelled(@Observes @Cancelled OrderEvent event) { ... }
```

### Observer Priority
```java
public void onOrderPlaced(@Observes(priority = 1) OrderPlacedEvent event) { ... }
public void auditOrder(@Observes(priority = 2) OrderPlacedEvent event) { ... }
```

### Transaction-Bound Observers
Observers can be tied to transaction lifecycle phases — built-in, no custom implementation needed.

```java
public void onOrderPlaced(@Observes(during = TransactionPhase.AFTER_SUCCESS) OrderPlacedEvent event) {
    // Only runs if the transaction committed successfully
}
```

Available `TransactionPhase` values: `IN_PROGRESS`, `BEFORE_COMPLETION`, `AFTER_COMPLETION`, `AFTER_FAILURE`, `AFTER_SUCCESS`.

---

## Feature Comparison

| Feature               | Spring                        | Quarkus/CDI                        |
|-----------------------|-------------------------------|------------------------------------|
| Publish               | `ApplicationEventPublisher`   | `Event<T>.fire()`                  |
| Subscribe             | `@EventListener`              | `@Observes`                        |
| Async publish         | `@Async` + `@EventListener`   | `Event<T>.fireAsync()` / `@ObservesAsync` |
| Filter events         | SpEL `condition=`             | CDI `@Qualifier` annotations       |
| Ordering              | `@Order`                      | `@Observes(priority=...)`          |
| Transaction hooks     | ❌ Needs custom implementation | ✅ Built-in via `TransactionPhase` |
| Specification         | Spring-proprietary            | Jakarta CDI standard               |

---

## Key Takeaway

Quarkus CDI events are generally more powerful than Spring's equivalent, particularly due to:
- **Qualifier-based filtering** for fine-grained event routing
- **Built-in transaction phase support** without extra boilerplate
- Being based on the **open Jakarta CDI standard**
