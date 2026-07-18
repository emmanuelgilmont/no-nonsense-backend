# Chapter 8 — Consuming a public API without overthinking it

> 📦 **Source code:** [`weather-service`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/spring-boot/weather-service)
> (Spring Boot) and [`q-weather`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/quarkus/q-weather)
> (Quarkus) in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. You want to know the weather for a Belgian town — maybe "Leuven," maybe
just the postal code "3000" — and turn that into something a person can read, or
something another service (like Chapter 7's Discord bot) can push out as a
notification. There's a free API for this, [Open-Meteo](https://open-meteo.com/), no
key required. The naive version is "call it, return the JSON." The actual work is
everything around that one call: resolving a name to coordinates, deciding what to do
when the name doesn't resolve, not hammering the upstream API for the same answer
twice, and turning a JSON blob into something readable.

This chapter covers the Spring Boot version in depth, and then asks a more interesting
question at the end: why does an identical service exist twice, in two different
frameworks?

## The naive fix

The obvious version calls Open-Meteo's geocoding endpoint, takes the first result, then
calls the forecast endpoint with those coordinates:

```java
var coords = geocode(placeName);       // first result, whatever it is
return forecast(coords.lat, coords.lon);
```

Two problems show up almost immediately. First, "the first result" isn't always the
right one — search for a 4-digit Belgian postal code and Open-Meteo's geocoding API
doesn't treat that specially; you need to actually check whether a candidate result
carries that postcode, not just trust result ordering. Second, calling both endpoints on
every single request means every request pays the same latency and the same load on a
free public API, for information that hasn't changed in the last few minutes (a
forecast) or the last few years (where a town is).

## What actually happens

### Resolving a place, correctly

```java
@Cacheable(cacheNames = CacheConfig.CACHE_GEOCODING, key = "#query")
public Optional<ResolvedPlace> geocode(String query) {
    JsonNode root = geoClient.get()
        .uri(uri -> uri.path("/search")
            .queryParam("name", query)
            .queryParam("count", 10)
            .queryParam("language", props.language())
            .queryParam("format", "json")
            .queryParam("countryCode", props.countryCode())
            .build())
        .retrieve()
        .body(JsonNode.class);
    // ... filter to the configured country, then:

    String postcodeWanted = extractPostcode(query).orElse(null);
    if (postcodeWanted != null) {
        for (JsonNode n : results) {
            if (hasPostcode(n, postcodeWanted))
                return Optional.of(toResolved(n, postcodeWanted));
        }
    }
    // otherwise the first result (Open-Meteo generally orders by relevance)
    return Optional.of(toResolved(results.get(0), postcodeWanted));
}
```

[`OpenMeteoClient.java#L61-L102`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/infra/OpenMeteoClient.java#L61-L102)

The query is asked for up to 10 candidates, not just one, precisely so a postal code
embedded in the query (extracted with a small regex, `\b(\d{4})\b`) can be matched
against a specific candidate's postcode list rather than blindly trusting whichever
result the API happens to rank first. Only when no postcode is present — a plain city
name — does the code fall back to "first result, trust the API's own ranking."

### A fallback that only fires where it's safe to

```java
private ResolvedPlace resolvePlaceWithFallback(String place) {
    var first = geocode(place);
    if (first.isPresent())
        return first.get();

    // fallback only if we're on the defaultQuery (otherwise it'd surprise the caller)
    if (place.equalsIgnoreCase(props.defaultQuery())) {
        for (String fb : props.fallback()) {
            var r = geocode(fb);
            if (r.isPresent())
                return r.get();
        }
    }
    throw new NoSuchElementException("Place introuvable: " + place);
}
```

[`OpenMeteoClient.java#L45-L59`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/infra/OpenMeteoClient.java#L45-L59)

This is the one detail in the whole project worth slowing down on. A list of fallback
locations exists in configuration, and it would be tempting to apply it whenever
*any* place fails to resolve — "be helpful, always try to return something." The code
deliberately doesn't do that: fallback only triggers when the failed query was already
the configured *default* location. Silently substituting a different place when someone
explicitly asks for one that doesn't resolve would return a weather report for the wrong
town without saying so — worse than a clear error, because it looks correct. Falling
back only for the default query keeps the "just give me something reasonable" behavior
for `/weather/default` while a genuinely wrong place name still fails loudly for
anyone who typed one on purpose.

### Two caches, deliberately separate TTLs

```java
public static final String CACHE_GEOCODING = "geocoding";
public static final String CACHE_FORECAST  = "forecast";

@Bean CacheManager cacheManager(WeatherProperties props) {
    var geo = new CaffeineCache(CACHE_GEOCODING,
        Caffeine.newBuilder().expireAfterWrite(props.cache().geocodingTtl()).maximumSize(10_000).build());
    var forecast = new CaffeineCache(CACHE_FORECAST,
        Caffeine.newBuilder().expireAfterWrite(props.cache().forecastTtl()).maximumSize(10_000).build());
    // ...
}
```

[`CacheConfig.java#L16-L32`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/config/CacheConfig.java#L16-L32)

Geocoding ("where is Leuven") and forecasting ("what's the weather at these
coordinates") are cached separately, with separately configured TTLs — 7 days by
default for geocoding, 15 minutes for forecasts, per `application.conf`. That gap is the
point: a place's coordinates are effectively permanent, but a forecast is stale within
the hour. Caching both under one TTL would either throw away a geocoding result far too
early or serve an hour-old forecast as if it were current — same shape of trade-off as
Chapter 2's Redis cache, at in-process Caffeine scale instead.

### Errors that reach the client mean something

```java
@ExceptionHandler(NoSuchElementException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ProblemDetail notFound(NoSuchElementException ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    pd.setTitle("Place not found");
    pd.setDetail(ex.getMessage());
    return pd;
}

@ExceptionHandler(Exception.class)
@ResponseStatus(HttpStatus.BAD_GATEWAY)
public ProblemDetail badGateway(Exception ex) {
    var pd = ProblemDetail.forStatus(HttpStatus.BAD_GATEWAY);
    pd.setTitle("Upstream error");
    pd.setDetail(ex.getMessage());
    return pd;
}
```

[`ApiExceptionHandler.java#L17-L35`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/handler/ApiExceptionHandler.java#L17-L35)

Worth contrasting directly with Chapter 7: this project *does* translate failure into
something meaningful. A place that genuinely can't be resolved is a client-facing 404,
using [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) `ProblemDetail` rather than a
raw stack trace; anything else — Open-Meteo unreachable, a malformed response — becomes
a 502 Bad Gateway, correctly signaling "the problem is upstream, not with your request."
What isn't here, and would be the natural next step in a less personal setting, is any
retry or circuit-breaker behavior around Open-Meteo itself — a transient blip still
surfaces immediately as a 502 rather than being quietly retried once.

## A small, deliberate piece of plumbing: HOCON config

Configuration for this service lives in `application.conf`, not
`application.properties` — [HOCON](https://github.com/lightbend/config), the format
popularized by Typesafe/Lightbend Config, chosen for nested structure and comments that
plain `.properties` doesn't give you as cleanly. Spring Boot doesn't understand `.conf`
files out of the box, so a small custom loader bridges the two:

```java
public class HoconPropertySourceLoader implements PropertySourceLoader {
    @Override
    public String[] getFileExtensions() {
        return new String[]{"conf"};
    }

    @Override
    public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
        Config cfg = ConfigFactory.parseReader(
            new InputStreamReader(resource.getInputStream(), StandardCharsets.UTF_8)
        ).resolve();

        Properties props = new Properties();
        for (Map.Entry<String, ConfigValue> e : cfg.entrySet()) {
            props.put(e.getKey(), stringify(e.getValue().unwrapped()));
        }
        return List.of(new PropertiesPropertySource(name, props));
    }
}
```

[`HoconPropertySourceLoader.java#L17-L38`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/config/HoconPropertySourceLoader.java#L17-L38)

It implements Spring Boot's `PropertySourceLoader` SPI, parses the HOCON file with the
real Typesafe Config library, flattens every resolved value into a plain
`java.util.Properties`, and hands that back to Spring as an ordinary property source —
registered via `spring.factories`, the exact same auto-configuration mechanism Chapter 1
used for the transaction-token starter. From that point on, `@ConfigurationProperties`
binds to it exactly as if it had come from a `.properties` file, because as far as Spring
is concerned, it did.

## The pieces that make this work

**[`WeatherController`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/controler/WeatherController.java#L15-L39)**
— three thin `@GetMapping` endpoints, `@Validated` bounds on `days` (`@Min(1) @Max(3)`).
Note the package name, `controler` — a real typo in the source, left as-is, in the same
harmless spirit as `discord-service`'s `PublicController` hosting the private-message
endpoint. Neither one is a design choice; both are the kind of thing that's obvious once
noticed and invisible before.

**[`WeatherService`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/service/WeatherService.java#L10-L53)**
— clamps `days` to the configured maximum, falls back to the default query when no
place is given, and builds the plain-text summary for `/weather/human` with a single
`String.format` template.

**[`OpenMeteoClient`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/infra/OpenMeteoClient.java#L20-L206)**
— geocoding, the postcode-aware matching, the conditional fallback, and the forecast
call that walks Open-Meteo's JSON response into a typed `WeatherResponse`. All of it
shown above.

**[`WMOWeatherCode`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/spring-boot/weather-service/src/main/java/be/gate25/weather/domain/WMOWeatherCode.java)**
— a lookup enum translating Open-Meteo's numeric WMO weather codes into human-readable
English descriptions. A dedicated i18n setup would be overkill for a table that's
small, stable, and English-only by choice.

## Why does the same service exist twice?

`weather-service` and `q-weather` do, deliberately, the exact same thing — same
endpoints, same Open-Meteo integration, same fallback and caching behavior, same
postcode-matching logic, even the same `controler` typo in the package name, carried
over from one project to the other. The reason for the second one isn't technical
necessity; it's that trying Quarkus at least once was worth doing on its own, and
re-implementing a service whose requirements were already fully understood was a
low-risk way to do it.

That said, building the same thing twice does surface real, concrete differences —
not opinions about which framework is "better," just what changes when the same logic
is expressed in a different framework's idioms.

**Configuration gets simpler.** The Quarkus version has no HOCON loader at all — plain
`application.properties`, because there was no equivalent need invented for it. That's
not a limitation of Quarkus; it's one fewer piece of custom plumbing to carry, in a
version built after the first one already existed.

**The REST client becomes declarative.** Where the Spring Boot version builds a
`RestClient` and calls `.get().uri(...).retrieve()` imperatively inside
`OpenMeteoClient`, the Quarkus version declares the upstream contract as interfaces and
lets the framework generate the implementation:

```java
@Path("/")
@RegisterRestClient(configKey = "geocoding-api")
public interface GeocodingClient {
    @GET
    @Path("/search")
    @Produces(MediaType.APPLICATION_JSON)
    JsonNode search(@QueryParam("name") String name, @QueryParam("count") int count,
        @QueryParam("language") String language, @QueryParam("format") String format,
        @QueryParam("countryCode") String countryCode);
}
```

[`GeocodingClient.java#L13-L21`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/quarkus/q-weather/src/main/java/be/gate25/weather/infra/GeocodingClient.java#L13-L21)

MicroProfile REST Client's `@RegisterRestClient` turns that interface into a working HTTP
client with no method body at all — the URL comes from `configKey`, the parameters map
straight to query params. It's a genuinely different way of expressing "call this
endpoint": contract-first instead of built-by-hand, and worth having actually written
once to feel the difference rather than just read about it.

**Startup footprint is the real reason it runs in production.** The Spring Boot version
is the reference implementation — it's what the README documents in the most
detail — but the Quarkus version is what actually runs on the homelab Docker host, for
the ordinary reason Quarkus exists: a small, fast-booting JVM footprint for a service
that does very little and should start almost instantly.

**And one honest gap the port left behind.** Comparing the two `OpenMeteoClient`
implementations line for line surfaces something the README for `q-weather` doesn't
mention: the geocoding cache annotation is commented out.

```java
// @CacheResult(cacheName = "geocoding")
public Optional<ResolvedPlace> geocode(String query) {
    JsonNode root = geoClient.search(query, 10, props.language(), "json", props.countryCode());
    // ...
```

[`OpenMeteoClient.java#L65-L67`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/quarkus/q-weather/src/main/java/be/gate25/weather/infra/OpenMeteoClient.java#L65-L67)

The forecast cache, just below it in the same file, is active
([`@CacheResult(cacheName = "forecast")`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/quarkus/q-weather/src/main/java/be/gate25/weather/infra/OpenMeteoClient.java#L98)) —
only the geocoding one is disabled. The most likely explanation, reconstructed from the
code rather than confirmed: this is leftover from porting the Spring Boot version, where
the equivalent line is a live `@Cacheable`. Somewhere in translating that to Quarkus's
`@CacheResult`, the annotation was commented out — maybe to debug something, maybe
mid-refactor — and never re-enabled. The practical effect is small (geocoding results
are cheap to recompute and geocoding rarely runs as often as forecasting does) but it's
a genuine, undocumented discrepancy between what the README claims ("Caches geocoding
results (7 days)... with Caffeine") and what the running code actually does. Worth
fixing; also worth naming rather than quietly cleaning up before anyone noticed it was
ever a gap.

## Interview questions worth being ready for

**"Why maintain two implementations of the same service?"**
Not a production need — one service would do. The Spring Boot version came first and is
the one the documentation is built around; the Quarkus version exists because trying
Quarkus at least once, on a problem already fully understood, was a deliberately
low-risk way to learn its idioms. It happens to also be the version actually deployed,
for an ordinary reason: smaller footprint, faster boot, for a service that does very
little.

**"Why not apply the fallback location whenever any place fails to resolve, since
you already have the list?"**
Because that would substitute a different, unrequested location for one the caller
explicitly typed, and return a plausible-looking result instead of a clear error. That's
strictly worse than failing loudly — a wrong answer that looks right is harder to catch
than an honest 404. Restricting the fallback to only the configured default query keeps
"give me *something* reasonable" working for `/weather/default` without ever silently
answering a different question than the one that was asked.

**"Why two separate caches instead of one?"**
Because they invalidate on completely different timescales. A place's coordinates don't
meaningfully change; a forecast is stale within the hour. A single shared TTL would be
wrong for at least one of the two — too short for geocoding, or too long for forecasts —
no matter which value you picked.
