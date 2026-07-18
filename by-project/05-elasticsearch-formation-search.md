# Chapter 5 — Finding a needle in a NAS full of documents

> 📦 **Source code:** [`elasticsearch-formation-search`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/elasticsearch/elasticsearch-formation-search)
> in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. You've got a NAS with a few thousand PDFs, slide decks, and videos,
accumulated over years of online courses and webinars. Somewhere in there is a document
that explains the one config property you need right now. You know it exists. You do not
know which folder it's in, what it's called, or whether the phrase you remember is even
the phrase that was used.

`grep -r` across a NAS full of binary PDFs and MP4 metadata gets you nowhere. This
chapter is about the layer that actually makes that content searchable — and about the
point where Spring Data's convenience stops being convenient and you have to drop down to
something more explicit.

## The naive fix (and why it stops working almost immediately)

Spring Data Elasticsearch lets you get a repository for free, the same way Spring Data
JPA does — declare an interface, name your methods well, and the query writes itself:

```java
public interface FormationRepository extends ElasticsearchRepository<FormationDocument, String> {
    List<FormationDocument> findByFileExtension(String extension);
    List<FormationDocument> findByMetaLanguage(String language);
}
```

[`FormationRepository.java#L8-L12`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/repository/FormationRepository.java#L8-L12)

This is genuinely useful, and it's still in the codebase. The temptation is to keep
going the same way for the actual search feature — something like
`findByContentContaining(String keyword)` — and for a while it would even work.

It falls apart the moment the requirements stop being "does this field contain this
value" and become "rank documents by relevance, across two fields, with the title
counting more than the body, and show me *why* each result matched." Method-name
derivation can express filters. It cannot express a boosted multi-field relevance
query, and it cannot express highlighting. There's no method name for "count a title
match three times more than a content match." That's not a Spring Data limitation to
work around — it's the actual difference between "look this up" and "search this,"
and it's the reason the rest of this chapter exists.

## What actually happens

```
REST client
    │
    ▼
GET /v1/search?q=&extension=&from=&to=
    │
SearchController
    │
SearchService
    ├── FormationRepository      (kept for the simple lookups)
    └── ElasticsearchOperations  (full-text, boost, highlight, aggregations)
    │
    ▼
Elasticsearch (FSCrawler index: formations)
```

1. A request lands on `/v1/search` with any combination of a free-text query, an
   extension filter, and a date range — all optional.
2. `SearchService` builds a `BoolQuery` piece by piece: a full-text `multiMatch` clause
   in the `must` position if `q` was supplied, and `term`/`range` clauses in the
   `filter` position for extension and date.
3. The query runs through `ElasticsearchOperations`, with a highlight request attached,
   against the `formations` index that FSCrawler maintains independently of this
   service.
4. Each hit is mapped to a flat `SearchResult` record, pulling the matched excerpt out
   of the highlight fields if one exists.

## Two fields, one score: the boost

```java
if (request.q() != null && !request.q().isBlank()) {
    bool.must(Query.of(q -> q
        .multiMatch(m -> m
            .query(request.q())
            .fields("meta.title^3", "content")
        )
    ));
}
```

[`SearchService.java#L38-L45`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L38-L45)

`meta.title^3` is the whole trick: a match in the title counts three times more toward
the relevance score than the same match in the document body. Searching "spring boot"
puts a document *titled* "Introduction to Spring Boot" ahead of a five-hundred-page PDF
that happens to mention the phrase once on page 340 — without either document being
excluded, just ranked differently. This is the kind of thing a SQL `LIKE` query has no
concept of at all: a relational database can tell you a row matches, not how well it
matches relative to another row.

## `must` vs `filter`: two clauses that look similar and aren't

The extension and date-range conditions don't go through `multiMatch` — they go into
the query's `filter` context instead:

```java
if (request.extension() != null) {
    bool.filter(Query.of(q -> q
        .term(t -> t.field("file.extension").value(request.extension()))
    ));
}
```

[`SearchService.java#L48-L55`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L48-L55)

This distinction is easy to miss and worth naming explicitly: `must` clauses
contribute to the relevance score; `filter` clauses only decide yes-or-no membership and
are cached by Elasticsearch since their result doesn't depend on the rest of the query.
Putting the extension filter in `must` would technically still return correct results —
but it would let "does this happen to be a PDF" quietly influence how documents are
*ranked* against each other, which makes no domain sense, and it would throw away the
caching Elasticsearch gives filter clauses for free. The fix here isn't a bug fix, it's
knowing which of two structurally similar clauses to reach for and why.

## Highlighting: showing your work

```java
private HighlightQuery highlight() {
    return new HighlightQuery(
        new Highlight(List.of(new HighlightField("content"))),
        FormationDocument.class
    );
}
```

[`SearchService.java#L129-L134`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L129-L134)

Returning a relevance score is only half the point of full-text search — the other half
is letting the person see *why* a document matched without opening it. The highlight
query rides alongside the main query and comes back attached to each hit; `toResult()`
pulls the first highlighted fragment out and drops it into the `highlight` field of the
response:

```java
String snippet = hit.getHighlightFields()
    .getOrDefault("content", List.of())
    .stream()
    .findFirst()
    .orElse(null);
```

[`SearchService.java#L136-L153`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L136-L153)

That's the line that turns a JSON array of document IDs into something a person can
actually scan.

## The `/paths` endpoint, and the bucket you have to filter back out

`/v1/paths` is meant to answer one question: which folders exist in the index? The
naive implementation is "aggregate on `path.virtual.tree` and return every bucket key" —
and it almost works, except FSCrawler indexes the path field as a tree of tokens, and
the *deepest* token for any file is the file's own full path, not a folder:

```
/webinaires/Limiteless/cours.mp4
  → /webinaires
  → /webinaires/Limiteless
  → /webinaires/Limiteless/cours.mp4
```

Aggregate on that field without any further thought and every individual file shows up
in the bucket list right alongside the actual folders — `/paths` would return both
`/webinaires/Limiteless` and `/webinaires/Limiteless/cours.mp4` as if they were peers,
which defeats the point of an endpoint whose whole job is "list the folders, not the
files":

```java
return aggs.get("paths")
    .aggregation()
    .getAggregate()
    .sterms()
    .buckets()
    .array()
    .stream()
    .map(b -> b.key().stringValue())
    .filter(p -> !p.matches(".*/[^/]+\\.[^/]+$"))
    .sorted()
    .toList();
```

[`SearchService.java#L99-L125`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L99-L125)

The regex is the actual fix: it excludes any bucket key that looks like
`.../something.extension` — a leaf — and keeps only the folder-shaped keys. Nothing in
FSCrawler's indexing is wrong here; the field genuinely does what it's documented to do
(support directory aggregation *and* full-path lookups from the same field). The service
layer is where the two use cases get separated back out.

## The pieces that make this work

**[`SearchController`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/controller/SearchController.java#L20-L40)**
— three thin `@GetMapping` endpoints. All optional parameters, all delegation, no query
logic — that all lives one layer down.

**[`FormationDocument`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/document/FormationDocument.java#L13-L64)**
— the `@Document`-mapped shape of what FSCrawler puts into the index. Note
`createIndex = false`: this service reads an index FSCrawler owns and maintains; it has
no business creating or reshaping it.

**[`SearchService`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/service/SearchService.java#L26-L154)**
— everything above lives here: the `BoolQuery` assembly, the boost, the `must`/`filter`
split, the highlight wiring, and the aggregation-with-a-regex-filter for `/paths`.

**[`FormationRepository`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/main/java/be/gate25/search/repository/FormationRepository.java#L8-L12)**
— kept deliberately, not replaced. It's the right tool for the two lookups that really
are just "match a field."

## Testing strategy — one layer, on purpose

Every other project in this portfolio splits tests into a fast local layer (H2,
Caffeine — no Docker) and a Docker-gated layer that exercises the real infrastructure.
This project doesn't, and that's a deliberate exception rather than an oversight: there
is no in-memory stand-in for Elasticsearch's query DSL the way H2 stands in for
PostgreSQL or Caffeine stands in for Redis. A `multiMatch` boost, a highlight, and a
`path.virtual.tree` aggregation are Elasticsearch-specific behavior — testing them
against anything other than real Elasticsearch would mean testing something else
entirely.

```java
@SpringBootTest
@Testcontainers(disabledWithoutDocker = true)
class SearchServiceIntegrationTest {

    @Container
    @ServiceConnection
    static ElasticsearchContainer es = new ElasticsearchContainer(
        "docker.elastic.co/elasticsearch/elasticsearch:8.19.8"
    ).withEnv("xpack.security.enabled", "false");
```

[`SearchServiceIntegrationTest.java#L24-L33`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/test/java/be/gate25/search/SearchServiceIntegrationTest.java#L24-L33)

Three documents are seeded once for the whole class
([`SearchServiceIntegrationTest.java#L41-L52`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/main/elasticsearch/elasticsearch-formation-search/src/test/java/be/gate25/search/SearchServiceIntegrationTest.java#L41-L52)),
and each test asserts against real Elasticsearch behavior: a keyword match returns the
right document, an extension filter narrows to exactly one, a date range includes what
it should, `/browse` scopes correctly under a path prefix, and `/paths` simply doesn't
throw. That last assertion is admittedly thin — it confirms the aggregation-and-regex
pipeline runs without error, not that the regex excludes exactly the right buckets on a
larger, more varied path tree. Worth strengthening with an explicit assertion on the
returned folder list if this project grows past three seeded documents.

## Interview questions worth being ready for

**"Why keep the repository at all if `ElasticsearchOperations` can do everything?"**
Because it can't do everything *more clearly*. `findByFileExtension` reads instantly;
the equivalent hand-built `TermQuery` is three lines of builder for no extra
expressiveness. The rule of thumb this project follows: reach for the repository until
you need something it structurally cannot express — relevance scoring, boosting,
highlighting, aggregations — and only then drop to the explicit API, for exactly the
part that needs it.

**"Why Elasticsearch instead of PostgreSQL full-text search, given the rest of the
portfolio already uses PostgreSQL?"**
Postgres's built-in text search (`tsvector`/`tsquery`) can do basic ranking, and it's a
fair option at a smaller scale. Where it stops being competitive is exactly the two
things this chapter leans on: field-weighted relevance across a document corpus that's
growing indefinitely, and generating highlighted excerpts as part of the query itself
rather than as a separate post-processing step. Elasticsearch is purpose-built for that;
Postgres full-text search is a capability bolted onto a system whose primary job is
transactional row storage. If the actual need were "filter structured rows with an
occasional text match," Postgres would be the better and simpler choice — this project
exists because the need is closer to "rank fuzzy, unstructured documents by relevance."

**"What happens if FSCrawler changes the index mapping — does this service break
silently?"**
It would, and that's a real gap worth naming rather than glossing over: `createIndex =
false` means this service trusts FSCrawler's mapping completely and has no schema
validation of its own. A renamed field on the FSCrawler side (say, `file.last_modified`
becoming `file.lastModified`) would make the corresponding filter silently return zero
matches rather than fail loudly, because Elasticsearch doesn't error on querying a field
that doesn't exist — it just doesn't match anything. There's no contract test between
the two systems here; in a production setting I'd want one, probably a scheduled check
that a known field is present and of the expected type before traffic is routed to this
service.

---

*Next: Chapter 6 — The anchor: why event-driven matters in finance (`kafka-financial-pipeline`)*
