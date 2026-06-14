# Java NIO, Netty & Reactive Programming — Concept Summary

## 1. The Core Problem: Thread-per-Connection

In traditional **blocking I/O** (e.g. Spring MVC on Tomcat), each incoming connection gets its own dedicated thread. That thread blocks while waiting for data, DB results, or any other I/O — doing nothing useful in the meantime. At high concurrency this becomes expensive: thousands of threads consume large amounts of memory and cause heavy context-switching overhead.

---

## 2. Java NIO Selector: 1 Thread, Many Connections

Java NIO introduces the `Selector`, which allows a **single thread to monitor multiple channels** (sockets) simultaneously. Instead of blocking on one channel, the thread calls `selector.select()` — which blocks until *any* of the registered channels is ready — and then handles only the ones that are ready.

Key components:

- **Selector** — the multiplexer that watches many channels
- **SelectableChannel** — a non-blocking channel (e.g. `SocketChannel`, `ServerSocketChannel`)
- **SelectionKey** — a token representing one channel's registration, carrying interest ops (`OP_ACCEPT`, `OP_READ`, `OP_WRITE`, `OP_CONNECT`) and ready ops

The thread is not truly "free" during `select()` — it is still blocked there. The difference is it blocks **in one place waiting for any of N channels**, rather than N threads each blocking on their own individual channel.

---

## 3. Netty: Boss Thread + Worker Threads

Netty builds on NIO and introduces a clean two-group architecture:

```
BossGroup (1 thread, 1 Selector)
  → ServerSocketChannel on port 8080
  → watches OP_ACCEPT only
  → when a client connects: accepts it, hands the SocketChannel to a Worker

WorkerGroup (n threads = CPU cores, n Selectors)
  → each Worker owns its own Selector and its own set of channels
  → watches OP_READ, OP_WRITE on established connections
  → runs your application handler code directly
```

Each channel is assigned to exactly one Worker EventLoop for its entire lifetime — it never moves. This means:

- **No shared state** between Worker threads at the Selector level
- **No locks or contention** on channel handling
- **Sequential, thread-safe** event handling per channel by design

---

## 4. Why Reactive Programming Is Still Needed

NIO/Netty solves the *thread-per-connection* problem, but not the *blocking application code* problem.

Netty uses only a **small fixed pool of Worker threads** (typically one per CPU core — e.g. 8 threads on an 8-core machine). If your handler blocks one of those threads on a synchronous DB call, you have frozen a significant fraction of the entire server's capacity.

| Model | Thread count | Driven by |
|---|---|---|
| Blocking (Tomcat) | ~200 (configurable) | Expected concurrent requests + I/O latency |
| Reactive (Netty) | CPU cores (e.g. 8) | Raw CPU capacity |

Reactive frameworks like **Project Reactor** (Spring WebFlux) and **Mutiny** (Quarkus) solve this by expressing application logic as **non-blocking chains**: the Worker thread is released while waiting for a DB response, picks up other work, and resumes when the result arrives.

```java
// Thread released between each step — never held hostage
Mono.fromCallable(() -> httpClient.get("/api/data"))
    .flatMap(response -> dbRepository.save(response))
    .map(result -> transform(result))
    .subscribe();
```

Reactive is **unnecessary in the blocking model** — since the thread is already committed to one request from start to finish, expressing it as a reactive chain gives no benefit.

---

## 5. The Cardinal Rule: Never Block the Event Loop

Because Worker threads are so few and so valuable:

- No synchronous DB calls (use R2DBC instead of JDBC)
- No `Thread.sleep()`
- No synchronous HTTP clients
- No CPU-heavy work inline (offload to a separate thread pool via `publishOn`)

```java
Mono.fromCallable(() -> fetchData())
    .publishOn(Schedulers.boundedElastic())  // offload to separate pool
    .map(data -> heavyCpuWork(data))
    .publishOn(Schedulers.parallel())        // back to event loop
    .map(result -> sendResponse(result));
```

This is the same rule as "don't block the event loop" in Node.js and Vert.x — same architecture, same constraint.

---

## 6. Multiple Sources: HTTP + Kafka

HTTP and Kafka fit different models and are handled independently:

| Source | Managed by | Selector involved? |
|---|---|---|
| Incoming HTTP | Netty Boss + Worker | Yes — OP_ACCEPT + OP_READ |
| Kafka messages | Kafka client poll thread (separate) | No — Kafka has its own protocol |
| Second TCP server | Second Netty Boss + shared Workers | Yes — separate Selector per Boss |
| Outbound HTTP calls | Netty client Worker | Yes — OP_CONNECT + OP_READ |

For **two TCP servers** (e.g. HTTP on 8080 + raw TCP on 9090), you use two `ServerBootstrap` instances each with their own Boss thread and `ServerSocketChannel`, but they typically **share the same Worker EventLoopGroup**.

Kafka polling runs on its own internal thread completely outside Netty, but in a reactive app its results are emitted as a reactive stream that can interact with the same pipelines as HTTP handlers.

---

## Key Takeaways

- **NIO Selector**: 1 thread monitors N channels — eliminates thread-per-connection
- **Netty Boss**: accepts connections (OP_ACCEPT), hands off to Workers
- **Netty Workers**: each owns a private Selector, handles I/O on assigned channels
- **Reactive model**: necessary because Workers are too few to ever block — application code must also be non-blocking
- **Selectors are network-only**: file I/O, Kafka, and other sources bring their own threading models
