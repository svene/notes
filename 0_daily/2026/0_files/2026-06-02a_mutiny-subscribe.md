# Mutiny (SmallRye) Subscription Best Practices

## Core principle: operators transform, endpoints subscribe

Never subscribe deep in your service layer. Return `Uni`/`Multi` up the call chain
and let the framework (Quarkus) handle the subscription at the endpoint level.

## Uni

- Emits at most one item, then terminates automatically
- Subscription is cleaned up on completion or failure
- No manual cancellation needed

## Multi

- Can be finite or infinite
- `.subscribe().with(...)` returns a `Cancellable` — do not discard it for long-lived streams
- For request-scoped streams, return the `Multi` to the endpoint handler

## Do: return up the chain

```java
class MyService {
    Multi<Item> getItems() {
        return multi.onItem().transform(item -> process(item));
    }
}

class MyResource {
    @GET
    @Path("/items")
    public Multi<Item> items() {
        return myService.getItems(); // Quarkus subscribes and manages cancellation
    }
}
```

## Don't: subscribe in the service layer

```java
class MyService {
    void doSomething() {
        multi.subscribe().with(item -> process(item)); // leaked Cancellable
    }
}
```

## Exception: background tasks

When there is no endpoint to return to (e.g. `@PostConstruct`, scheduled jobs),
subscribe manually and tie the `Cancellable` to the bean lifecycle:

```java
@ApplicationScoped
class MyService {
    private Cancellable cancellable;

    @PostConstruct
    void init() {
        cancellable = Multi.createFrom().ticks().every(Duration.ofSeconds(1))
            .subscribe().with(tick -> System.out.println(tick));
    }

    @PreDestroy
    void cleanup() {
        if (cancellable != null) cancellable.cancel();
    }
}
```

## What happens if you ignore a Cancellable on an infinite stream

- The stream runs forever, consuming resources
- The callback keeps executing with no way to stop it
- Multiple leaks accumulate if the subscribing code is called more than once
  (e.g. during Quarkus dev mode hot reload or repeated test runs)

## Endpoint response behavior for Multi

The response type depends on the media type declared on the endpoint.

### `application/json` — collected into a list

Quarkus waits for the `Multi` to complete, collects all items, and sends a single JSON array:

```java
@GET
@Produces(MediaType.APPLICATION_JSON)
public Multi<Item> items() { ... }
// → client receives: [{"id":1}, {"id":2}, {"id":3}]
```

### Streaming media types — items sent as they arrive

```java
@GET
@Produces(MediaType.SERVER_SENT_EVENTS)
public Multi<Item> items() { ... }
// → client receives SSE events pushed as they emit

@GET
@Produces("application/x-ndjson")
public Multi<Item> items() { ... }
// → client receives newline-delimited JSON, one object per line as they emit
```

### Warning: infinite Multi with application/json

Quarkus will wait forever trying to collect the list, effectively hanging the request.
Always use a streaming media type for infinite or long-running streams.

### Summary

| Media type              | Behaviour               | Use case                  |
|-------------------------|-------------------------|---------------------------|
| `application/json`      | Collected into array    | Finite result sets        |
| `application/x-ndjson`  | Streamed line by line   | Log tailing, data pipelines |
| `text/event-stream`     | Pushed as SSE events    | Real-time UI updates      |
