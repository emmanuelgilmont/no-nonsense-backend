# Chapter 3 — What happens between two threads when you call a service

> 📦 **Source code:** [`grpc-price-service`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/grpc/grpc-price-service)
> in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. Chapter 1 gave you a transaction token that follows every request
through MDC, with zero code changes anywhere in the call stack. It feels like magic —
until the call stack crosses a thread boundary that Spring didn't create for you.

That's exactly what happens the moment a REST controller delegates to a local gRPC
service. The HTTP request lands on a Tomcat thread. gRPC processes it on a Netty
thread. MDC is thread-local by design — it has to be, that's what makes it safe under
concurrent requests — which means the token you worked so hard to propagate
automatically in Chapter 1 doesn't cross this particular boundary automatically at
all. You have to do it by hand, once, in exactly the right place.

## The naive fix (and why it's tempting to stop too early)

The obvious instinct, once you know MDC is thread-local, is: "fine, I'll just copy the
value across when the gRPC call starts."

```java
@Override
public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {

    String token = headers.get(CORRELATION_KEY);
    MDC.put("transactionToken", token);          // set it once, here
    return next.startCall(call, headers);
}
```

This looks reasonable, and it's wrong in a way that a quick manual test won't catch.
`interceptCall()` sets up the call — it does not run the business logic. gRPC actually
processes the request later, inside the listener's `onHalfClose()` callback, which
fires *after* `interceptCall()` has already returned. Depending on the executor gRPC
uses to dispatch that callback, it may not even run on the same thread that executed
`interceptCall()`. A single `MDC.put()` at call-setup time can be gone — or simply
irrelevant to the thread that's about to log something — by the time
`PriceServiceImpl` actually runs.

This is the gRPC-specific cousin of the same lesson from Chapter 1: cleanup and
placement matter more than the mechanism itself. The fix there was "always clean up in
`finally`, because thread pools reuse threads." The fix here is "set the MDC value on
every callback that might run business logic, not once at call setup" — different
shape, same underlying discipline: don't assume a thread-local value survives past the
call that set it.

## What actually happens

```
  HTTP client
      │
      ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Spring Boot 3 — Hermes                                 │
  │                                                         │
  │  GET /price/{symbol}                                    │
  │       │                                                 │
  │  PriceController                                        │
  │  (Tomcat thread — MDC: transactionToken=uuid)           │
  │       │                                                 │
  │  MdcClientInterceptor ── x-correlation-id ──▶           │
  │       │                                    │            │
  │       ▼               gRPC (in-process)    │            │
  │  PriceServiceGrpc stub                     │            │
  │       │                          MdcServerInterceptor   │
  │       │                          (Netty thread)         │
  │       │                          MDC: transactionToken  │
  │       ▼                                    │            │
  │  PriceServiceImpl ◀────────────────────────┘            │
  │  (Netty thread — MDC: transactionToken=uuid)            │
  └─────────────────────────────────────────────────────────┘
```

1. A request comes in to `PriceController`. It's already running on a Tomcat thread
   whose MDC was populated by `TokenFilter` — the same starter from Chapter 1, reused
   here rather than reinvented.
2. `PriceController` calls the local gRPC service through a blocking stub, injected via
   `@GrpcClient`.
3. `MdcClientInterceptor`, registered globally with `@GrpcGlobalClientInterceptor`,
   intercepts the outgoing call, reads the token from MDC, and writes it into the gRPC
   call's `Metadata` under an `x-correlation-id` key — gRPC's equivalent of an HTTP
   header.
4. The call crosses into the gRPC/Netty world. `MdcServerInterceptor`, registered with
   `@GrpcGlobalServerInterceptor`, reads that metadata key and restores the token into
   MDC — but not once; on *every* listener callback that might do real work, because of
   the `onHalfClose()` timing problem above.
5. `PriceServiceImpl` runs on the Netty thread, logs normally with no token-handling
   code, looks up the price, and returns.
6. `PriceController` receives the response (or a `StatusRuntimeException`) back on the
   original Tomcat thread and turns it into JSON or an HTTP error.

Grep that token across the log file and you get both sides of the call — REST and
gRPC, two different thread pools — as one continuous story, exactly like the
cross-service case in Chapter 1, except here the "two services" are two layers of the
same JVM.

## What the developer actually writes

Same principle as the first two chapters: nothing.

```java
// PriceServiceImpl — no token-handling code at all
log.debug("Received GetPrice request for symbol: {}", symbol);
```

[`PriceServiceImpl.java#L25-L52`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/service/PriceServiceImpl.java#L25-L52)
is a plain gRPC service implementation. It doesn't know MDC exists, doesn't touch
`Metadata`, and doesn't know which thread it's running on. The two global interceptors
are the only place any of that logic lives — which is the same argument Chapter 1 made
for MDC over a custom logging wrapper: touch the plumbing once, not every call site.

## The pieces that make this work

**[`MdcClientInterceptor`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/interceptor/MdcClientInterceptor.java#L16-L41)**
— runs on the Tomcat thread, where MDC is already populated. It reads
`MDC.get(TokenFilter.MDC_TOKEN_KEY)` and writes it into the outbound call's headers
before the call starts, wrapping the client call so the write happens at the right
point in gRPC's own lifecycle (`ForwardingClientCall.SimpleForwardingClientCall`).

**[`MdcServerInterceptor`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/interceptor/MdcServerInterceptor.java#L17-L62)**
— the more interesting half. Rather than a single `MDC.put()` at `interceptCall()`
time, it wraps every listener callback (`onMessage`, `onHalfClose`, `onCancel`,
`onComplete`) with a small `withMdc()` helper that sets the token immediately before
the callback runs and removes it in a `finally` block immediately after. That's the
direct fix for the naive version above, and it's also the same "always clean up,
always in `finally`" discipline from Chapter 1's `TokenFilter`, just applied at
per-callback granularity instead of per-request.

**[`PriceController`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/controller/PriceController.java#L24-L57)**
— the REST → gRPC bridge. It injects a blocking stub with `@GrpcClient("price-service")`
and translates gRPC's `Status` codes back into HTTP status codes explicitly
(`NOT_FOUND` → 404, everything else → 500) rather than letting a raw
`StatusRuntimeException` leak to the client.

**[`PriceServiceImpl`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/service/PriceServiceImpl.java#L14-L53)**
— the actual gRPC endpoint, `@GrpcService`-annotated so `grpc-spring-boot-starter`
wires it up automatically. Business logic only: look up a price, return it, or signal
`NOT_FOUND`.

## Why a domain model that knows nothing about the wire

[`Price.java`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/model/Price.java#L9-L38)
is a plain class — not the proto-generated `PriceResponse`. `PriceServiceImpl` builds a
`PriceResponse` from a `Price` explicitly, field by field
([`PriceServiceImpl.java#L42-L47`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/service/PriceServiceImpl.java#L42-L47)),
rather than passing the generated class straight through from repository to wire.

It's a few extra lines for a stub with three hardcoded prices, and it would be fair to
call it slightly over-engineered at this scale. The reason to do it anyway: the
proto-generated classes belong to the transport contract, not to the domain. If the
`.proto` file changes shape — a renamed field, a new required value — that change
shouldn't ripple into repository or business logic that has nothing to do with gRPC.
[`PriceRepository`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/main/java/be/gate25/grpc/repository/PriceRepository.java#L13-L25)
only ever knows about `Price`.

## Testing strategy — two layers, split by what actually needs Spring

Same principle as the rest of the portfolio — separate what can be tested fast and in
isolation from what genuinely needs the full application context — applied here at a
finer grain than the Docker/no-Docker split from Chapter 2, because the thing that
needs isolating this time isn't infrastructure, it's Spring Boot auto-configuration
itself.

**[`PriceServiceIntegrationTest`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/237c8f07a716fe716a8615719949cc84cb86ff80/grpc/grpc-price-service/src/test/java/be/gate25/grpc/PriceServiceIntegrationTest.java#L35-L92)**
spins up a real gRPC server and channel using
`InProcessServerBuilder`/`InProcessChannelBuilder` — a genuine gRPC server, wired with
the real `PriceServiceImpl`, communicating in-memory instead of over a socket, with no
Spring context loaded at all.

| Scenario | Assertion |
|---|---|
| Known symbol (`BEL20:UCB`) | Response has a positive price, `EUR` currency, non-blank timestamp |
| Unknown symbol | `StatusRuntimeException` with status code `NOT_FOUND` |

That's genuinely fast — no container, no Spring Boot startup, no network — and it
exercises the real gRPC serialization path (unlike a plain unit test that would call
`priceServiceImpl.getPrice(...)` directly and skip the wire entirely). What it
deliberately doesn't cover: the interceptors. `MdcClientInterceptor` and
`MdcServerInterceptor` are registered globally via Spring Boot auto-configuration
(`@GrpcGlobalClientInterceptor` / `@GrpcGlobalServerInterceptor`), and an in-process
test with no Spring context never puts either interceptor on the call path.

**`PriceServiceMdcPropagationIT`** closes that gap on purpose. It's a full
`@SpringBootTest` on a random port — the one test in this module that pays the cost of
a real Spring Boot startup — specifically because MDC propagation only exists as a
property of the wired-up application, not of `PriceServiceImpl` in isolation. It sets
a known token on the calling thread, drives a real REST call through
`PriceController`, and asserts that the same token was present in MDC while
`PriceServiceImpl` ran on the Netty thread. That's precisely the naive-fix trap from
earlier in this chapter, turned into a regression test: if a future change reverted
`MdcServerInterceptor` back to a single `MDC.put()` at `interceptCall()` time instead
of per-callback, this is the test that would catch it — the in-process test above
cannot, structurally, no matter how thorough its other assertions are.

Two tests, two different jobs: one proves the gRPC contract behaves correctly and
stays fast enough to run constantly; the other proves the one property that can only
be observed with the full application wired together, and accepts the Spring Boot
startup cost as the price for testing it honestly rather than mocking it away.

## Interview questions worth being ready for

**"Why not just use a `ThreadLocal` yourself instead of MDC?"**
MDC *is* a `ThreadLocal` under the hood — SLF4J's `MDC` class is a thin,
logging-framework-aware wrapper around one. Using it directly, rather than rolling a
custom one, means every log line automatically renders it via the logging pattern
(`%X{transactionToken}`), and it's the same mechanism already established in Chapter 1
— no second convention to learn for gRPC specifically.

**"Why intercept on every listener callback instead of once, if the token doesn't
change during the call?"**
Because thread-locals don't travel with logical requests, they travel with physical
threads — and gRPC's callback dispatch doesn't guarantee `onHalfClose()` runs
synchronously on the thread that returned from `interceptCall()`. Setting it once at
call setup is a bet on an implementation detail of gRPC's executor that isn't part of
its contract. Setting and clearing it around each callback removes that bet entirely,
at the cost of a few extra `MDC.put()`/`MDC.remove()` calls per request — cheap
insurance for something that would otherwise fail silently and intermittently, which
is the worst kind of bug to chase.

**"Why gRPC at all for a call that never leaves the JVM?"**
Fair pushback — for a single-JVM demo, a plain method call would obviously work and
skip the whole problem this chapter is about. The point of building it this way is
that the pattern is realistic for services that *do* leave the JVM: an internal gRPC
service mesh where a REST-facing gateway fronts several backend services communicating
over gRPC for performance. Building the in-process version first is how you learn what
the thread-boundary problem looks like before adding a second machine and a real
network on top of it.

**"REST → gRPC bridge — isn't that an odd direction? Why not gRPC all the way to the
client?"**
Because most external clients — browsers, mobile apps, third-party integrations —
speak HTTP/JSON, not gRPC's binary framing, and gRPC isn't something you can poke with
a browser or `curl` the way you can REST. The bridge pattern here (`PriceController`
translating REST in, gRPC internally) is a reasonable shape for an API gateway that
needs to stay reachable by ordinary HTTP clients while its backend services talk gRPC
to each other.

---

*Next: [Chapter 4 — Processing yesterday's trades, one chunk at a time](./04-trade-settlement-batch.md)*
