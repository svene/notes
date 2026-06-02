# SmallRye Mutiny — API Summary

## The Two Layers

Mutiny has two distinct API layers that work together:

**Low-level: `subscribe()`**
The terminal operation that kicks off execution. Nothing runs until you call it.

```java
uni.subscribe().with(
    item -> System.out.println("got: " + item),
    failure -> System.err.println("failed: " + failure)
);
```

**High-level: `onItem()`, `onFailure()`, `onCompletion()`, etc.**
Pipeline builders — declarative, lazy, inert. They describe transformations but don't execute anything.

```java
uni.onItem().transform(x -> x * 2)
   .onFailure().recoverWithItem(-1)
   .subscribe().with(item -> ...);
```

**Mental model: `on*()` builds the pipe, `subscribe()` opens the tap.**

---

## Laziness — Nothing Runs Until You Subscribe

Mutiny is fully lazy. The `on*()` chain is like defining a recipe; `subscribe()` is cooking it.

```java
Uni<Integer> pipeline =
    someUni
        .onItem().transform(x -> x * 2)   // defined, not running
        .onFailure().recoverWithItem(-1);  // defined, not running

pipeline.subscribe().with(item -> System.out.println(item)); // NOW it runs
```

This is directly analogous to Java Streams: `.filter()` / `.map()` are lazy, `.collect()` triggers execution.

---

## The API Layers in Order

```
subscribe().with(...)           ← terminal, triggers execution
    ↑
onItem().transform()            ← transform items
onItem().transformToUni()       ← async transform (returns Uni/Multi)
onFailure().recoverWithItem()   ← error handling
    ↑
onItem().invoke()               ← sync side effects (logging, metrics)
onItem().call()                 ← async side effects
    ↑
plug() / stage()                ← operator composition helpers
```

---

## What Goes in the Pipe vs. `with()`

| Put in the pipe (`on*()`) | Put in `with()` |
|---|---|
| Transformations | Handling the final result |
| Error recovery | Handling the final failure |
| Side effects (logging, metrics) | Writing HTTP response, sending to Kafka, etc. |
| Reusable, composable logic | Caller-specific endpoint logic |

The pipe is **reusable and composable**. `with()` is the **specific endpoint** for a particular call site.

```java
// Reusable pipeline — can be returned from a method, injected, shared:
Uni<Order> processedOrder = rawOrderUni
    .onItem().transform(this::validate)
    .onItem().transformToUni(this::enrich)
    .onFailure().recoverWithItem(Order.empty());

// Caller A:
processedOrder.subscribe().with(
    order -> httpResponse.send(order.toJson()),
    failure -> httpResponse.sendError(500)
);

// Caller B:
processedOrder.subscribe().with(
    order -> kafkaProducer.send(order),
    failure -> alerting.notify(failure)
);
```

---

## Can You Subscribe Without `with()`?

Technically yes, but with caveats:

```java
// Explicit no-op subscriber
uni.subscribe().withSubscriber(UniSubscriber.noop());

// Returns CompletableFuture (discard if you don't care)
uni.subscribe().asCompletionStage();
```

However, always provide at minimum a **failure handler**. Subscribing without one causes Mutiny to throw if a failure occurs, which can be surprising:

```java
// Dangerous — swallows failures silently or throws unexpectedly:
uni.subscribe().with(item -> {});

// Safe minimum for fire-and-forget:
uni.subscribe().with(item -> {}, failure -> log.error("failed", failure));
```

---

## Quick Reference: When to Use What

| Situation | API |
|---|---|
| Transform items synchronously | `onItem().transform()` |
| Async transform (returns Uni/Multi) | `onItem().transformToUni()` / `transformToMulti()` |
| Sync side effects (logging, metrics) | `onItem().invoke()` |
| Async side effects | `onItem().call()` |
| Error recovery | `onFailure().recoverWithItem()` / `.retry()` |
| Trigger execution + handle result | `subscribe().with(item -> ..., failure -> ...)` |
| Block for result (testing/legacy) | `await().atMost(Duration)` |
| Custom backpressure / framework use | `subscribe().withSubscriber(...)` |
| Collect Multi into list | `multi.subscribe().asList()` |
| Convert Multi to Java Stream | `multi.subscribe().asStream()` |

Note: `asList()` and `asStream()` are terminal — they subscribe internally despite looking like transformations.

---

## Best Practices

**1. Always provide a failure handler when subscribing.**
Omitting it leads to unhandled failures that can silently disappear or throw unexpectedly at runtime.
```java
// Always do this:
uni.subscribe().with(item -> handle(item), failure -> log.error(failure));
```

**2. Keep the pipe reusable; keep `with()` caller-specific.**
Build your `Uni`/`Multi` pipeline as a reusable object. Only at the call site do you subscribe with the specific handlers for that context (HTTP, Kafka, tests, etc.).

**3. Use `invoke()` for side effects in the pipe, not `with()`.**
If you're logging or recording metrics mid-pipeline, `onItem().invoke()` is the right place. Reserve `with()` for the final result consumption.

**4. Never write `subscribe().with(item -> {})` with an empty lambda.**
This is a code smell. Either you care about the result (put a real handler) or you care only about side effects (put them in `invoke()` in the pipe). An empty `with()` usually means the logic is in the wrong place.

**5. Prefer `on*()` APIs over raw `subscribe().withSubscriber()`.**
The raw `Subscriber` API is for framework authors and custom backpressure scenarios. For application code, the `on*()` fluent API is safer, more readable, and handles the subscription lifecycle for you.

**6. Remember that `asList()` and `asStream()` on Multi are terminal.**
They look like transformations but they subscribe internally. Don't chain further `on*()` calls after them.

**7. Use `await().atMost()` in tests, not blocking hacks.**
Mutiny provides first-class blocking support for tests. Don't manually block threads.
```java
Order result = processedOrder.await().atMost(Duration.ofSeconds(5));
```
