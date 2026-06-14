# Virtual Threads vs. Vert.x: Is Reactive Still Justified?

## 1. Is Vert.x Still Justified With Virtual Threads?

Virtual threads (Java 21+) allow blocking-style code to scale like non-blocking code. The JVM
transparently unmounts a virtual thread from its carrier OS thread when it blocks on I/O —
conceptually similar to what Vert.x does manually with epoll. This directly challenges Vert.x's
core value proposition.

### Head-to-Head Comparison

| Concern | Vert.x / Mutiny | Virtual Threads |
|---|---|---|
| Scalability (I/O bound) | ✅ Excellent | ✅ Excellent |
| Code readability | ❌ Reactive pipelines are hard | ✅ Plain imperative Java |
| Debugging & stack traces | ❌ Painful | ✅ Normal stack traces |
| Learning curve | ❌ Steep | ✅ Minimal |
| CPU-bound workloads | ⚠️ No benefit | ⚠️ No benefit either |
| Memory per "thread" | ✅ Very low (event loop) | ✅ Very low (~few KB) |
| Pinning risk | N/A | ✅ Fixed in JDK 24/25 |

### Where Vert.x / Reactive Still Wins

- **Backpressure** — Virtual threads have no built-in answer. Reactive streams with `Multi`
  give explicit flow control; virtual threads would just create thousands of them under a burst.
- **Explicit async composition** — Complex pipelines (fan-out, merging streams, conditional
  branching) are easier to express declaratively in Mutiny than with imperative virtual thread
  code using locks and queues.
- **Resource control** — Vert.x gives precise control over concurrency; virtual thread
  scheduling is JVM-managed and harder to reason about under load.
- **Existing reactive ecosystem** — A Quarkus + Mutiny + Kafka + MongoDB reactive stack is
  already tuned and tested together.

### Quarkus Supports Both

```java
@GET
@RunOnVirtualThread  // blocking style is fine here
public String myEndpoint() {
    Document doc = mongoCollection.find(filter).first();
    return doc.toJson();
}
```

The pragmatic approach: **virtual threads for REST endpoints and simple flows, reactive/Mutiny
for Kafka stream processing where backpressure matters**. Quarkus supports mixing both.

---

## 2. Backpressure: Mutiny or Vert.x?

Both layers play a role — they are complementary, not alternatives.

| Layer | Backpressure Role |
|---|---|
| **Mutiny** | *Decides* when to apply backpressure — coordinates the `request(n)` signalling protocol |
| **Vert.x** | *Enforces* it — pauses/resumes actual socket I/O when Mutiny signals |
| **TCP** | Propagates it end-to-end back to the remote sender |

```
Kafka broker / MongoDB
        │
        │  TCP flow control ← Vert.x pauses socket reads
        │
      Netty
        │
      Vert.x           ← enforces the pause at I/O level
        │
      Mutiny           ← coordinates the request(n) signals
        │
   your pipeline       ← you express what you want via operators
```

Neither alone is sufficient — Mutiny without Vert.x's socket control would buffer endlessly;
Vert.x without Mutiny's signalling would have no way to know when to pause.

---

## 3. Ecosystem Maturity in 2026 (Based on Current Research)

### The Pinning Problem: Solved

The biggest historical concern — `synchronized` blocks silently pinning virtual threads to
carrier threads — was fixed in JDK 24 via **JEP 491** (March 2025), carried forward into
JDK 25 LTS. Most projects are now free from pinning, and the JDK authors no longer recommend
replacing `synchronized` with `ReentrantLock`.

### Library Compatibility: Largely Caught Up

- Apache HTTP Client 5.4 replaced `synchronized` with Java lock primitives explicitly for
  virtual thread compatibility.
- Spring Boot now recommends virtual threads and supports them via a simple property.
- Tooling (JFR, jcmd, IntelliJ) now fully supports virtual thread diagnostics.

### The Remaining Concern: ThreadLocal

Popular libraries like Jackson and Netty use `ThreadLocal` as an object pooling mechanism,
assuming a limited, recycled thread pool. With virtual threads — which are disposable and
short-lived — this causes massive allocation and memory pressure. Quarkus therefore still
uses an explicit opt-in model rather than enabling virtual threads globally.

### Updated Maturity Table

| Concern | Earlier Assessment | 2026 Reality |
|---|---|---|
| Pinning on `synchronized` | ⚠️ Real risk | ✅ Fixed in JDK 24/25 |
| Library compatibility | ⚠️ Patchy | ✅ Mostly resolved |
| ThreadLocal memory pressure | ⚠️ Watch out | ⚠️ Still a real concern |
| Tooling (debuggers, profilers) | ⚠️ Immature | ✅ Fully supported |
| Reactive still needed for Kafka/backpressure | ✅ Yes | ✅ Still yes |

### The 2026 Verdict

If you have an I/O-bound service on Java 25 and have not enabled virtual threads yet, you are
leaving throughput and simplicity on the table for no good reason. If you have a CPU-bound
service or a reactive stack with backpressure needs, virtual threads are still not the answer.
Industry leaders predicted 70% adoption of virtual threads by 2027, with the expected pattern
being **hybrids: virtual threads for core logic, reactive for Kafka and WebSocket edge cases**.
