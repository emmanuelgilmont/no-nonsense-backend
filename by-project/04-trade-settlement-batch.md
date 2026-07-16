# Chapter 4 — Processing yesterday's trades, one chunk at a time

> 📦 **Source code:** [`trade-settlement-batch`](https://github.com/emmanuelgilmont/java-backend-portfolio/tree/main/batch/trade-settlement-batch)
> in the [java-backend-portfolio](https://github.com/emmanuelgilmont/java-backend-portfolio) repo.

Picture this. It's 6 PM. The trading desk is done for the day. Somewhere, a process
needs to take every trade that happened today, work out what actually needs to be
settled, and produce two things: a row in a database and a line in a report file that
someone downstream is going to read tomorrow morning. Nobody's watching it run. If it
dies at 2 AM halfway through fifty thousand trades, you need to know exactly how far it
got — not "did it work," but "how many rows, which ones, and can I safely run it again."

This is the classic shape of a batch job, and it's different enough from a REST request
that the tools change. A REST request either succeeds or it doesn't, and the client is
sitting there waiting. A batch job processes a huge number of records with nobody
watching, and "restart safely from where it broke" isn't a nice-to-have, it's the whole
point.

## The naive fix (and why it falls apart around record ten thousand)

The obvious first instinct is a loop:

```java
List<TradeRecord> trades = readAllTradesFromCsv(inputFile);
List<Settlement> settlements = new ArrayList<>();

for (TradeRecord trade : trades) {
    if (!"CANCELLED".equals(trade.getStatus())) {
        settlements.add(computeSettlement(trade));
    }
}

jdbcTemplate.batchUpdate(insertSql, settlements);
writeCsv(outputFile, settlements);
```

For five rows in a test CSV, this is indistinguishable from the real thing. That's
exactly what makes it a trap — it works right up until it doesn't, and the failure
modes only show up at production scale:

- **Memory.** `readAllTradesFromCsv` loads every trade into a `List` at once. Fine for
  five rows, not fine for a real EOD file with hundreds of thousands.
- **All-or-nothing.** Everything happens in one giant unit of work. If the process dies
  after processing 40,000 of 50,000 trades, you don't have "40,000 done" — you have
  nothing committed, or everything committed with no record of where the crash
  happened, depending on how carefully you wired the transaction.
- **No restart story.** Run it again and you either duplicate every row you already
  wrote, or you have to build your own bookkeeping to figure out what's already done.
- **No numbers.** How many rows were read? How many were skipped? How many actually got
  written? You'd have to add counters by hand, and now you're rebuilding, badly, a
  chunk of what a batch framework gives you for free.

None of this is visible from the code. It reads like ordinary Java. That's the same
lesson as Chapters 1 through 3 in a new shape: the plumbing that makes something
production-safe is invisible until you go looking for it, or until it's 2 AM and you
don't have it.

## What actually happens

Spring Batch's answer is **chunk-oriented processing**: read one item, process one item,
repeat until you've accumulated a fixed-size chunk — then write the whole chunk and
commit, in one transaction, before moving to the next chunk.

```
trades_YYYYMMDD.csv
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Spring Batch — settlementJob                       │
│                                                     │
│  FlatFileItemReader<TradeRecord>                    │
│      │  (1 item per CSV row, skip header)           │
│      ▼                                              │
│  SettlementProcessor                                │
│      │  CANCELLED → null (filtered, no write)       │
│      │  PENDING   → Settlement (T+2, qty × price)   │
│      ▼                                              │
│  CompositeItemWriter<Settlement>                    │
│      ├── JdbcBatchItemWriter → settlements table    │
│      └── FlatFileItemWriter  → settlements_*.csv    │
└─────────────────────────────────────────────────────┘
```

1. The `tradeReader` reads one CSV row and maps it to a `TradeRecord`. It doesn't hold
   the whole file in memory — it reads on demand, one record at a time, until the file
   is exhausted.
2. `SettlementProcessor` runs on that single record. CANCELLED trades come back as
   `null`, which is Spring Batch's built-in signal for "drop this item, don't write it,
   but count it." PENDING trades come back as a computed `Settlement`.
3. Spring Batch accumulates processed items until it has 10 of them — the configured
   **chunk size** — then hands the whole chunk to the writer in one go.
4. `CompositeItemWriter` fans that chunk out to two delegates: a `JdbcBatchItemWriter`
   that inserts into `settlements`, and a `FlatFileItemWriter` that appends to the daily
   CSV report.
5. Both writes for that chunk happen inside one transaction, managed by the
   `PlatformTransactionManager` Spring Boot wires in automatically. If either writer
   fails, the whole chunk rolls back — but the chunks that already committed before it
   stay committed.
6. The `JobRepository` — Spring Batch's own bookkeeping, backed by `BATCH_JOB_INSTANCE`,
   `BATCH_JOB_EXECUTION`, `BATCH_STEP_EXECUTION` tables — records everything about the
   run: how many items were read, filtered, written, and whether it finished or died,
   before you ever write a line of business logic.

That's the actual value chunking buys you: a crash at row 40,001 loses at most the
in-flight chunk of 10, not the other 40,000 already committed.

## What the developer actually writes

Same principle as every chapter so far — the framework carries the mechanics, the
method body stays about the business rule:

```java
@Override
public Settlement process(TradeRecord trade) {
    if ("CANCELLED".equalsIgnoreCase(trade.getStatus())) {
        return null; // Spring Batch: skip, don't write, count it in filterCount
    }

    BigDecimal settlementAmount = trade.getPrice()
            .multiply(BigDecimal.valueOf(trade.getQuantity()));

    Settlement settlement = new Settlement();
    settlement.setTradeId(trade.getTradeId());
    // ...
    settlement.setSettlementAmount(settlementAmount);
    settlement.setSettlementDate(trade.getTradeDate().plusDays(2)); // T+2
    settlement.setStatus(SettlementStatus.SETTLED);
    return settlement;
}
```

[`SettlementProcessor.java#L28-L57`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/processor/SettlementProcessor.java#L28-L57)

No transaction handling, no chunk counting, no thread management. Two lines carry the
entire business rule — the `null` return for CANCELLED trades
([`L30-L34`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/processor/SettlementProcessor.java#L30-L34))
and the settlement math
([`L36-L48`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/processor/SettlementProcessor.java#L36-L48)).
Everything about *how* that flows through a chunk, a transaction, and two writers lives
in configuration, not here.

One detail worth calling out on its own: `settlementAmount` and `price` are
[`BigDecimal`, never `double`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/domain/Settlement.java#L28-L31).
`0.1 + 0.2` in `double` is `0.30000000000000004` — a rounding error that's merely
annoying in most software and simply not acceptable when the number represents money
moving between counterparties.

## The pieces that make this work

**[`BatchConfig`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L50-L185)**
— every bean the job needs, in one place:

- [`settlementJob`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L57-L65)
  — the `Job` itself: one step, one completion listener.
- [`settlementStep`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L71-L83)
  — this is where `chunk(10, transactionManager)` actually gets declared: read →
  process → write, 10 items per transaction.
- [`tradeReader`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L89-L113)
  — a `FlatFileItemReader`, `linesToSkip(1)` to drop the CSV header, one
  `fieldSetMapper` lambda turning each row into a `TradeRecord`.
- [`dbWriter`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L124-L138)
  — a `JdbcBatchItemWriter` with `.beanMapped()`, so named parameters like `:tradeId`
  resolve straight from `Settlement`'s getters — no manual parameter binding.
- [`csvWriter`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L144-L169)
  — a `FlatFileItemWriter` building the output path from job parameters
  (`outputDir` + `runDate`).
- [`compositeWriter`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L175-L184)
  — wraps both writers in one `CompositeItemWriter`; for each chunk it calls the DB
  writer, then the CSV writer, so both destinations always reflect the same chunk.

**[`SettlementProcessor`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/processor/SettlementProcessor.java#L23-L58)**
— the business rule, shown above.

**[`JobCompletionListener`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/listener/JobCompletionListener.java#L19-L38)**
— fires once, after the job ends, and logs `readCount` / `writeCount` / `filterCount`
pulled straight off `JobExecution`. Those numbers already exist inside Spring Batch's
own bookkeeping; this listener just makes them visible in the log without your code
tracking a single counter by hand.

**Domain objects**
— [`TradeRecord`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/domain/TradeRecord.java#L15-L29)
(the input row) and
[`Settlement`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/domain/Settlement.java#L22-L39)
(the output row) are plain `@Data` (Lombok) containers, no logic. `SettlementStatus`
is a
[three-value enum](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/domain/SettlementStatus.java#L6-L13) —
worth noting honestly that only `SETTLED` is actually reachable today; `CANCELLED` is
filtered before a `Settlement` object even exists, and `FAILED` is reserved for error
handling that isn't implemented yet.

## Two annotations that will bite you if you get them backwards

**`@StepScope`** on `tradeReader` and `csvWriter` — both need a job parameter
(`inputFile`, or `outputDir` + `runDate`) that Spring doesn't know at application
startup, only at the moment the job is launched with real parameters. Without
`@StepScope`, Spring tries to build the bean at context-load time, before any parameter
exists, and the injection fails. `@StepScope` delays bean creation until the step
actually starts — by then the parameters are there.

**`@EnableBatchProcessing`** — the trap in the other direction. It's tempting to add
this annotation out of habit, especially coming from Spring Batch 4. In Spring Boot 3 /
Spring Batch 5 it does the opposite of what the name suggests: it *disables* Boot's
auto-configuration of the `JobRepository` and `PlatformTransactionManager`, and the
application fails to start. The rule here, as the class-level comment in
[`BatchConfig.java#L45-L48`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/java/be/gate25/batch/config/BatchConfig.java#L45-L48)
spells out, is simply: don't add it. Spring Boot already does this for you.

## Testing strategy — two layers

Same principle as Chapter 2's cache: **test behaviour locally, test infrastructure in
CI.** The dev machine here runs Windows without Docker — deliberately, to mirror an
environment where the batch can't just talk to a real PostgreSQL whenever it feels
like it.

### Layer 1 — `SettlementJobLocalTest` (local, no Docker)

`@ActiveProfiles("test")` swaps PostgreSQL for H2 in-memory
([`application-test.properties`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/test/resources/application-test.properties#L1-L24)) —
for both the `JobRepository`'s own `BATCH_*` tables and the business `settlements`
table. Same Spring Batch context, different `DataSource`, and the
`JdbcBatchItemWriter` genuinely can't tell the difference — it just speaks JDBC.

```java
JobExecution execution = jobLauncherTestUtils.launchJob(
        new JobParametersBuilder()
                .addString("inputFile", inputFile)
                .addString("runDate", "2026-05-21")
                .addString("outputDir", tempDir.toAbsolutePath().toString())
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters()
);

assertThat(execution.getStatus()).isEqualTo(BatchStatus.COMPLETED);

StepExecution step = execution.getStepExecutions().iterator().next();
assertThat(step.getReadCount()).isEqualTo(5);
assertThat(step.getFilterCount()).isEqualTo(1);  // T003 is CANCELLED
assertThat(step.getWriteCount()).isEqualTo(4);
```

[`SettlementJobLocalTest.java#L54-L76`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/test/java/be/gate25/batch/SettlementJobLocalTest.java#L54-L76)

Worth pointing out the `.addLong("timestamp", ...)` line — it's not incidental. Spring
Batch's `JobRepository` refuses to re-run a job with an identical set of parameters that
already completed (`JobInstanceAlreadyCompleteException`), which is exactly the
"don't process the same file twice" protection you want in production. In a test suite
that runs the same job repeatedly, that same protection would block every run after the
first, so the timestamp exists purely to make each test execution parametrically
unique — a deliberate, narrow workaround for the tests, not a loophole in the real
guarantee.

A second test in the same class checks the file side of the same run:

```java
Path csvOutput = tempDir.resolve("settlements_2026-05-21.csv");
assertThat(csvOutput).exists();
assertThat(java.nio.file.Files.lines(csvOutput).count()).isEqualTo(5); // header + 4 rows
```

[`SettlementJobLocalTest.java#L78-L97`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/test/java/be/gate25/batch/SettlementJobLocalTest.java#L78-L97)

### Layer 2 — `SettlementJobPostgresIT` (CI, Docker required)

`@Testcontainers(disabledWithoutDocker = true)` starts a real `postgres:16-alpine`
container on a random port; `@DynamicPropertySource` rewires the `DataSource` to point
at it before the Spring context loads:

```java
@Container
static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:16-alpine")
                .withDatabaseName("batchdb")
                .withUsername("batch")
                .withPassword("batch");

@DynamicPropertySource
static void overrideDataSource(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
}
```

[`SettlementJobPostgresIT.java#L46-L62`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/test/java/be/gate25/batch/SettlementJobPostgresIT.java#L46-L62)

What this layer adds on top of Layer 1 is the thing H2 structurally cannot prove: that
real PostgreSQL types and constraints behave the way the code assumes. The test doesn't
just check that the job completed — it queries PostgreSQL directly for facts H2's
in-memory approximation wouldn't catch a type mismatch on:

```java
Integer t2Count = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM settlements WHERE settlement_date = '2026-05-23'",
        Integer.class);
assertThat(t2Count).isEqualTo(4); // tradeDate 2026-05-21 + 2 days
```

[`SettlementJobPostgresIT.java#L79-L111`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/test/java/be/gate25/batch/SettlementJobPostgresIT.java#L79-L111)

| Aspect | H2 (Layer 1) | Real PostgreSQL (Layer 2) |
|---|---|---|
| Spring Batch BATCH_* tables | ✓ (H2 dialect) | ✓ (PostgreSQL dialect) |
| Settlements persisted | ✓ | ✓ |
| Real PostgreSQL DATE / TIMESTAMP | ✗ | ✓ |
| T+2 date queryable via SQL | ✗ | ✓ |

Both layers share the same
[`schema.sql`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/resources/db/schema.sql#L1-L16),
deliberately written in SQL standard enough to run unmodified on both H2 and
PostgreSQL — the same "same DDL, different engine" idea Chapter 2 used for Redis vs.
Caffeine.

### Why not just run it on the dev machine?

`spring.batch.job.enabled=false` in
[`application.properties`](https://github.com/emmanuelgilmont/java-backend-portfolio/blob/7b3a5898dc16dd36822251255fdf363f4cf01ecd/batch/trade-settlement-batch/src/main/resources/application.properties#L14-L15)
means the job never launches automatically on startup — it has to be triggered
explicitly, which matters once this is deployed: nobody wants a settlement batch firing
itself the moment a container restarts. In practice this also means the batch itself
never actually runs end-to-end on the Windows dev machine, which has no PostgreSQL and
no Docker. Locally, the only two commands that make sense are `mvn test` (Layer 1,
behaviour) and `mvn package` (produce the JAR). The real execution against real
PostgreSQL happens on the homelab server, where the built JAR is copied over and run
inside Docker Compose alongside the database. It's a constraint that forces the test
suite to be genuinely useful without infrastructure, rather than a workaround to route
around later.

## Interview questions worth being ready for

**"What's a chunk in Spring Batch, concretely?"**
The number of items processed inside a single transaction. The reader pulls items one
at a time, the processor transforms each one, and once the chunk size is reached — 10
here — the writer receives the whole chunk and everything commits together. If it fails
mid-chunk, that chunk rolls back; chunks already committed before it stay committed.
It's the mechanism that turns "did the whole job succeed" into "how far did it get,"
which is the entire reason to reach for a batch framework instead of a loop.

**"How do you handle bad or invalid data?"**
In this project, only one case is handled: CANCELLED trades, filtered by the processor
returning `null`, counted in `filterCount`. Real parsing failures or a downed database
aren't handled explicitly here — Spring Batch supports `skip()` and `retry()` policies
at the step level for exactly that, and I'd bring that up unprompted rather than let it
look like an oversight: it's a natural next step, not implemented in this version.

**"How do you stop the same batch running twice on the same data?"**
The `JobRepository` records every execution together with its exact job parameters. Ask
it to run again with the *same* parameters after a completed run, and it refuses with
`JobInstanceAlreadyCompleteException` instead of silently reprocessing and duplicating
rows. That's also precisely why the tests add a `timestamp` parameter — to make each
test run parametrically distinct on purpose, not to defeat the protection.

**"Why `@StepScope` on the reader and CSV writer, but not on the DB writer?"**
`@StepScope` exists to delay bean creation until job parameters are actually available.
`tradeReader` needs `inputFile`, `csvWriter` needs `outputDir` and `runDate` — both are
only known once the job launches with real parameters, not at application startup.
`dbWriter` doesn't reference any job parameter, only the `DataSource`, which is ready at
context-load time — so it doesn't need the same treatment. `compositeWriter` is
step-scoped anyway, because one of its two delegates is.

**"Why not `@EnableBatchProcessing`?"**
Because in Spring Boot 3 with Spring Batch 5 it does the opposite of what it sounds
like — it turns *off* Boot's automatic wiring of the `JobRepository` and
`PlatformTransactionManager`. It's a common trap for anyone carrying over habits from
Spring Batch 4. The fix is not adding it at all; Boot's auto-configuration already
covers it.

**"Why write to both a database and a CSV file for the same data?"**
Two different downstream consumers with two different needs: the database row is
queryable, the CSV is what an ops or back-office process picks up the next morning
without needing DB access. `CompositeItemWriter` keeps both in sync per chunk rather
than running two separate passes over the same data — if the DB write for a chunk
fails, the CSV write for that same chunk doesn't silently succeed on its own, because
both live inside the same chunk transaction.

---

*Next: Chapter 5 — Finding a needle in a NAS full of documents*
