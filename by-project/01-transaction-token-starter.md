# Chapter 1 — Why every log line should tell a story

> 📦 **Source code:** [`starters/transaction-token-starter`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/starters/transaction-token-starter)
> in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. Production breaks at 2 AM. A user reports an error. You open the log file
and find fifty requests interleaved in the same three seconds — Alice's request, Bob's
request, a background job, all mixed together, one line after another, with no way to
tell which line belongs to which request.

You start grepping for timestamps. You guess. You lose twenty minutes before you even
start fixing the actual bug.

This chapter is about a small, boring fix for that problem: giving every request its
own token, and making sure every log line carries it.

## The naive fix (and why it doesn't scale)

The obvious answer is to pass a token through every method call. Something like:

```java
public void processOrder(String orderId, String token) {
    log.info("[{}] Processing order {}", token, orderId);
    validateOrder(orderId, token);
    chargePayment(orderId, token);
}
```

It works. It's also miserable to maintain. Every method signature grows an extra
parameter that has nothing to do with what the method actually does. Add a new service
call three layers deep, and you're threading that token through code that shouldn't
care about it at all.

There's a better way, and Java has had the tool for it for a long time: **MDC** (Mapped
Diagnostic Context), part of SLF4J. Think of it as a small thread-local map. You put a
value in once, at the very start of the request, and every `log.xxx()` call on that
thread can see it — automatically, without passing anything anywhere.

## What actually happens

Here's the flow, end to end:

1. A request comes in. A servlet filter checks for an `X-Correlation-ID` header. If the
   caller sent one (because it's a downstream call from another service), reuse it. If
   not, generate a fresh UUID.
2. That token goes into MDC.
3. Every log statement for the rest of the request — in any class, any layer — includes
   the token automatically, because the logging pattern references `%X{transactionToken}`.
4. The response echoes the token back in the `X-Correlation-ID` header.
5. If something throws, the error response body includes the token too.
6. At the end of the request, the filter cleans up the MDC. Always. Even on failure.

That last point matters more than it looks. Thread pools reuse threads. If you don't
clean up, thread N might carry Alice's token into Bob's request the next time it's
picked from the pool — and now your logs lie to you in a way that's actively worse than
having no correlation at all.

## What the developer actually writes

Nothing. That's the point.

```java
log.info("Processing order {}", orderId);
log.debug("Step 1 complete");
throw new RuntimeException("Something went wrong");
```

No token parameter. No boilerplate. And the logs look like this:

```
14:32:01.123 [a1b2c3d4-e5f6-7890-abcd-ef1234567890] INFO  OrderService - Processing order 42
14:32:01.124 [a1b2c3d4-e5f6-7890-abcd-ef1234567890] DEBUG OrderService - Step 1 complete
14:32:01.125 [a1b2c3d4-e5f6-7890-abcd-ef1234567890] ERROR GlobalExceptionHandler - Unhandled exception
```

Every line for this request shares the same token. `grep` that token against the log
file and you get the entire transaction, start to finish, with nothing else mixed in.

If it fails, the client gets the token back in the error body:

```json
{
  "status": 500,
  "message": "An unexpected error occurred. Please contact the helpdesk with your token.",
  "token": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

Now the helpdesk doesn't need to ask "what time did it happen, roughly?" They grep the
token and they're looking at the exact trace in ten seconds.

## The four pieces that make this work

**[`TokenFilter`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/starters/transaction-token-starter/src/main/java/be/gate25/tokencontext/TokenFilter.java#L37-L58)**
— a servlet filter, first thing that touches the request. Reads or generates the token,
puts it in MDC, and — this is the part people forget — cleans it up in a `finally` block
once the response is done. Filters run for *every* request, success or failure, so this
is the right place to guarantee cleanup.

**[`TokenContext`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/starters/transaction-token-starter/src/main/java/be/gate25/tokencontext/TokenContext.java#L31-L33)**
— a thin static wrapper around MDC, for the rare case where you need the token value
itself rather than just having it show up in logs (for example, to pass it explicitly to
a downstream HTTP call).

**[`GlobalExceptionHandler`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/starters/transaction-token-starter/src/main/java/be/gate25/tokencontext/GlobalExceptionHandler.java#L34-L56)**
— a `@RestControllerAdvice` that catches unhandled exceptions, logs the stack trace
server-side, and returns a clean error body with the token — without leaking internal
details to the client.

**[`TokenContextAutoConfiguration`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/starters/transaction-token-starter/src/main/java/be/gate25/tokencontext/TokenContextAutoConfiguration.java#L28-L48)**
— the Spring Boot auto-configuration class that wires all of this in automatically the
moment you add the dependency. No `@Import`, no manual bean registration.

That's genuinely all there is to it. Four small classes, one thread-local map, and log
correlation stops being a manual, error-prone exercise.

## Where this gets interesting: microservices

If service A calls service B, you want the *same* token to appear in both services'
logs. That's why the filter checks for an incoming `X-Correlation-ID` header before
generating a new UUID — and why the outgoing call should set that header explicitly:

```java
restClient.get()
    .uri("http://service-b/api/something")
    .header("X-Correlation-ID", TokenContext.getToken())
    .retrieve()
    ...
```

Now a single `grep` across both services' log files reconstructs the entire
cross-service trace. This is also the exact idea I reuse later in the gRPC chapter — the
mechanics change (HTTP headers become gRPC metadata, and the thread boundary is trickier),
but the underlying goal doesn't: one token, one story, no matter how many hops the
request takes.

## A word on virtual threads

If you're on Spring Boot with `spring.threads.virtual.enabled=true`, MDC still works
correctly for normal request handling — each virtual thread carries its own MDC context.
The one place to be careful is if you manually spawn child threads inside a request. MDC
doesn't cross thread boundaries by itself; you have to copy it explicitly with
`MDC.getCopyOfContextMap()`. I ran into exactly this problem later, in a different shape,
when I built the gRPC interceptors — more on that in Chapter 3.

## Try it yourself

Clone the project, run the example app, and hit all three scenarios:

```bash
# Happy path
curl -v http://localhost:8080/api/greet?name=Alice

# 400 — token still shows up in the error body
curl http://localhost:8080/api/fail-gracefully

# 500 — token still shows up, stack trace stays server-side
curl http://localhost:8080/api/fail-hard
```

Then open the log file and confirm: every line for a given request shares one token,
and different requests never mix.

## Interview questions worth being ready for

**"Why not just use a request ID library instead of rolling your own?"**
Fair question. This one is deliberately small and dependency-free — it's a Spring Boot
Starter you drop in with zero configuration, and it's a good example of what a
`spring.factories` / auto-configuration setup actually looks like under the hood. In a
real team, I'd evaluate existing solutions (Sleuth/Micrometer Tracing, for instance)
before reinventing this. But building it once, from scratch, is how you actually
understand what those bigger libraries are doing for you.

**"What happens if the client sends a malicious or malformed `X-Correlation-ID`?"**
Worth thinking about — right now the filter trusts whatever string arrives. In
production you'd want to validate the format (UUID shape) before trusting it, mostly
to avoid log injection or absurdly long values polluting your log lines.

**"Why MDC and not just a custom logging wrapper?"**
Because every existing `log.info(...)` call in the codebase — yours and any library's —
already goes through SLF4J. MDC piggybacks on infrastructure that's already there. A
custom wrapper would mean touching every call site, which defeats the entire point.

---

*Next: [Chapter 2 — Why you cache, and what can go wrong when you do](./02-fx-rate-service.md)*
