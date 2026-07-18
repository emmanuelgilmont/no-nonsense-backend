# Chapter 7 — Talking to Discord from Java, for fun and automation

> 📦 **Source code:** [`discord-service`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/spring-boot/discord-service)
> in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. You want a small, personal automation: every morning, a cron job checks
the weather and drops a message in a Discord channel. Or you want a way to page yourself
when something on your homelab breaks. None of that needs a Discord bot in the usual
sense — no slash commands, no reacting to messages, no listening to anything. It just
needs to *send* something, occasionally, on demand.

That distinction is the whole chapter. Once you separate "a bot that talks" from "a bot
that listens," the right tool changes completely.

## The decision that shapes everything else: no bot framework

Discord's most common Java tooling — [JDA](https://github.com/discord-jda/JDA),
Discord4J, Javacord — all exist to build a *real* bot: something that opens a persistent
WebSocket connection to Discord's gateway, stays connected, and reacts to events as they
arrive. That's the right tool if you're building a bot that listens for commands or
reacts to messages. It's the wrong tool if all you ever do is push a notification
outward.

`discord-service` doesn't use any of them. It uses `RestTemplate` — plain Spring HTTP,
already in the classpath, zero extra dependency — to call Discord's REST API directly:

```java
private final RestTemplate restTemplate = new RestTemplate();

@Value("${discord.token}")
private String TOKEN;
```

[`DiscordService.java#L13-L21`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/service/DiscordService.java#L13-L21)

The reasoning, honestly reconstructed after the fact rather than something written down
at the time: this service never needs to *receive* anything from Discord. No gateway
connection, no persistent state, no event loop — just an outbound `POST` whenever
something needs to be said. A bot framework built around a WebSocket gateway buys you
automatic rate-limit handling, reconnection logic, and a rich event model — none of
which this project has any use for. `RestTemplate` costs nothing extra and does exactly
the two things this service needs: open a DM channel, send a message.

| | `RestTemplate` (chosen here) | A bot framework (JDA, Discord4J…) |
|---|---|---|
| Dependencies | None — already in Spring | A real library, plus its own transport stack |
| Connection model | Stateless HTTP call, per request | Persistent WebSocket to Discord's gateway |
| Natural fit | "Push a notification from a backend service" | "Build an interactive bot that reacts to events" |
| Rate limiting | Not handled — you're on your own | Handled automatically, per-route |
| Retry / reconnection | Not handled | Built in |

The first row of that table is really the whole story: no dependency, and no need to
maintain a gateway connection you'd never use.

## What the service actually does

Three endpoints, one underlying service class:

```
POST /v1/discord/private?userId=...&message=...   # DM one user, or several (comma-separated)
POST /v1/discord/public?message=...                # post to a pre-configured channel
GET  /v1/discord/ping                              # DM the admin, to confirm the bot is alive
```

Sending a DM is two Discord API calls, not one — a detail the project's own README
doesn't spell out, because it's invisible from the outside:

```java
public void sendPrivateMessage(String userId, String message) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Bot " + TOKEN);
    headers.setContentType(MediaType.APPLICATION_JSON);

    String[] users = userId.split(",");
    for (String user : users) {
        // Step 1: open the direct message channel
        Map<String, String> dmBody = Map.of("recipient_id", user.trim());
        HttpEntity<Map<String, String>> dmRequest = new HttpEntity<>(dmBody, headers);

        ResponseEntity<Map> dmResponse = restTemplate
            .postForEntity("https://discord.com/api/v10/users/@me/channels", dmRequest, Map.class);

        String dmChannelId = (String) dmResponse.getBody().get("id");

        // Step 2: send the message
        Map<String, String> msgBody = Map.of("content", message);
        HttpEntity<Map<String, String>> msgRequest = new HttpEntity<>(msgBody, headers);

        restTemplate
            .postForEntity("https://discord.com/api/v10/channels/" + dmChannelId + "/messages", msgRequest, String.class);
    }
}
```

[`DiscordService.java#L47-L70`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/service/DiscordService.java#L47-L70)

A DM in Discord's model isn't sent to a user directly — there's no such endpoint. You
first open (or reuse) a private channel with that user via `POST
/users/@me/channels`, get back a channel ID, and *then* post a message to that channel,
exactly like posting to any other channel. It's the same primitive — "post a message to
a channel ID" — used twice, once to set the stage and once to actually say something.
The comma-separated `userId` parameter just loops that two-step dance once per
recipient.

A public message skips the channel-opening step entirely, since the channel ID is
already known and configured:

```java
public void sendPublicMessage(String message) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", "Bot " + TOKEN);
    headers.setContentType(MediaType.APPLICATION_JSON);

    Map<String, String> body = Map.of("content", message);
    HttpEntity<Map<String, String>> request = new HttpEntity<>(body, headers);

    restTemplate.postForEntity("https://discord.com/api/v10/channels/" + CHANNEL_ID + "/messages", request, String.class);
}
```

[`DiscordService.java#L28-L37`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/service/DiscordService.java#L28-L37)

## The pieces that make this work

**[`DiscordService`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/service/DiscordService.java#L13-L71)**
— all the Discord-facing logic: public messages, private messages, the two-step DM
dance. Everything else in the project is a thin REST wrapper around this class.

**[`PublicController`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/controller/PublicController.java#L11-L45)**
— worth naming honestly: despite the name, this class hosts *both* `/discord/public`
and `/discord/private`. It's a naming leftover, not a design decision — a small,
harmless inconsistency of the kind that's easy to miss once a project has more than one
controller and nobody goes back to rename things.

**[`HeartbeatController`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/discord-service/src/main/java/be/gate25/discord/controller/HeartbeatController.java#L11-L34)**
— `/discord/ping` doesn't do anything special; it just calls `sendPrivateMessage` with
the configured admin user and a fixed string. Liveness check, reusing the exact same
mechanism as any other DM.

**Configuration** — `DISCORD_TOKEN`, `DISCORD_CHANNEL_ID`, `DISCORD_ADMIN`, all injected
as environment variables, nothing hardcoded in source. Small detail, worth keeping: a
bot token in source control is a real, common way personal projects leak credentials.

## What this project deliberately doesn't do

This is the part worth being upfront about rather than glossing over. There is no retry,
no backoff, and no error handling at all around the calls to Discord's API. If Discord
returns a 429 (rate limited), a 401 (bad or revoked token), or a 5xx, or if the call
simply times out, `RestTemplate` throws, and that exception propagates straight up to
the HTTP client of `discord-service` as a generic 500. Nothing logs it specifically,
nothing retries it, nothing tells you *why* it failed beyond a stack trace.

Compare that to Chapter 1's `GlobalExceptionHandler`, or Chapter 8's `ApiExceptionHandler`
in `weather-service` — both projects that talk to an external API and both bother to
translate upstream failure into something a caller can actually reason about.
`discord-service` doesn't, and that's a real, honest gap, not a style choice presented
after the fact as intentional.

The actual reason is simpler and more mundane: this service exists to push a weather
notification to a personal Discord channel once a day. The blast radius of a failed call
is "I didn't get my weather message this morning," not lost data or a broken
transaction. At that scale, and for a solo project, the cost of building retry logic
didn't seem to buy anything real.

And it genuinely wouldn't, as it turns out — because the failure mode doesn't stop at
this service's boundary. The whole pipeline (Chapter 8's `weather-service`, this
project, and the daily notification) is glued together by a small bash script: `curl`
the weather text, then `curl` this service to post it. If either call fails, the script
does *nothing* — no retry, no logging, no alert. Even a perfectly retry-hardened
`discord-service` wouldn't fix that, because the actual single point of failure sits one
layer up, in an orchestrator that was never built to notice failure at all. Writing a
robust retry loop in Bash is technically possible and not remotely worth it here — the
honest fix, if this ever mattered more, would be replacing the orchestration itself, not
patching one link in a three-link chain that fails silently at every link.

## Interview questions worth being ready for

**"Why RestTemplate and not a proper Discord bot library?"**
Because this service never listens for anything — it only ever pushes a message
outward. A bot framework's whole value proposition is a maintained gateway connection
and an event model for *reacting* to Discord; neither is needed here. `RestTemplate`
against Discord's plain REST API does the two things this project actually needs — open
a DM channel, post a message — with zero extra dependency weight.

**"What happens if the Discord API call fails?"**
Right now: an unhandled exception surfaces as a generic 500, with nothing retried and
nothing specifically logged. That's a real gap, not a hidden feature. It's acceptable
here because the actual orchestration around this service (a bash script with no
failure handling of its own) wouldn't benefit from it anyway — the weak link is one
layer up. In a context where a missed message actually mattered, I'd fix the
orchestrator before I'd add retry logic to this one service in isolation.

**"Why open a DM channel every time instead of caching the channel ID per user?"**
Fair optimization to raise — Discord channel IDs for a given user pair are stable, so
repeated messages to the same user re-open a channel that already exists (which Discord
allows; it just returns the existing channel ID). For the traffic this project actually
sees — a handful of messages a day — the extra round trip is immaterial. It would be
worth caching if this were sending DMs at any real volume.

---

*Next: [Chapter 8 — Consuming a public API without overthinking it](./08-weather-service.md)*
