---
title: "From 70 Million Inserts to 25 Thousand: Batching Bulk Ingestion in Spring Boot"
published: false
description: How to ingest millions of CSV rows from object storage into PostgreSQL without melting a shared database — streaming parse, bounded batches, the 65,535-parameter limit, cached sequence allocation, and idempotent re-runs.
tags: springboot, java, postgres, performance
canonical_url: https://aayushmiglani.github.io/posts/batch-bulk-ingestion-spring-jpa/
cover_image:
---

> This article is also available on my blog: [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/batch-bulk-ingestion-spring-jpa/)

## The naive version, and why it melted the database

The task sounds boring until you see the row counts. Once a day, an upstream system drops a large CSV into object storage — call it `items-2026-05-24.csv`. Each line is one *item*, and every item carries up to 24 months of historical snapshots that you also need to persist as child rows. So one file is two tables: a parent `item` table and a child `item_history` table with a roughly 1-to-24 fan-out.

The first implementation is the one everybody writes. Stream the file, parse each line into a POJO, map it to JPA entities, and save:

```java
// The version that works in dev and dies in prod.
try (var reader = new InputStreamReader(objectStore.open(key))) {
    for (CsvLine line : parse(reader)) {
        Item item = toItem(line);
        item.setHistory(toHistory(line));   // up to 24 child rows
        itemRepository.save(item);          // cascades to item_history
    }
}
```

On a 5,000-row test file this is instant. On the real file it is a catastrophe. Walk through what each `save()` actually costs:

- **One `INSERT` per row, for both tables.** Three million items become three million parent inserts. The history cascade — up to 24 children each — becomes up to **72 million** child inserts. Hibernate sends them one statement at a time.
- **One sequence round-trip per insert.** Every entity needs a primary key, and with the default sequence strategy Hibernate issues a `SELECT nextval(...)` before each insert. That is *another* ~75 million round-trips, interleaved with the inserts.
- **An ever-growing persistence context.** Every saved entity stays managed in the Hibernate session. By row two million the session is holding millions of objects, dirty-checking all of them on every flush, and the JVM is fighting the garbage collector.

Add it up: on the order of **150 million** network round-trips to the database for a single file, each one a tiny transaction of work wrapped in real latency. On a dedicated database that is merely slow. On a *shared* database — one that other services are trying to serve live traffic from — it is an outage. The ingestion pegged database CPU for the duration and every unrelated query slowed down with it.

> **What you'll build.** A bulk-ingestion pipeline for Spring Boot + JPA that turns a multi-million-row file into a few tens of thousands of round-trips: a streaming CSV reader that never buffers the whole file, an accumulator that flushes fixed-size batches, a batch size derived from PostgreSQL's hard limit on bind parameters, sequence allocation that hands out IDs from memory, and an idempotent upsert so a half-finished run is safe to retry.

The shape we are aiming for:

```
  object storage                  bounded in-memory batch              PostgreSQL
  ┌────────────┐   stream    ┌──────────────────────────┐   flush   ┌──────────────┐
  │ items.csv  │ ─ line ───▶ │  [ item, item, item ...] │ ───────▶  │  multi-row   │
  │ (millions) │   by line   │   ≤ 120 parents          │           │  INSERT      │
  └────────────┘             │   + their history rows   │           └──────────────┘
                             └──────────────────────────┘                 │
                                        │  flush() + clear()               │
                                        ▼  (drop the batch, free memory)   ▼
                              session stays small              IDs pre-allocated in
                              regardless of file size          blocks, not per row
```

## Fix 1 — Stream and parse without buffering the file

The first rule of large-file processing: never hold the whole file in memory. The file lives in object storage (S3, GCS, Azure Blob — the SDK shape is the same), and every SDK gives you an `InputStream` rather than forcing you to download the object first. Read it as a stream and the JVM only ever holds the bytes it is currently parsing.

The parser matters more than people expect. The convenient choice is a binding library that reflects each row straight into an annotated bean. That convenience is expensive at this scale: per-row reflection and a fresh object graph for every line generate enormous allocation pressure. A **streaming, index- or name-based reader** — the kind that hands you one lightweight record at a time and lets you read fields by column name — parses far faster and allocates far less. (In the JVM world, FastCSV is a good example; the principle holds for any low-allocation streaming reader.)

```java
// Stream from object storage, parse lazily, never materialise the file.
try (InputStream in = objectStore.open(key);
     CsvReader<NamedCsvRecord> csv = CsvReader.builder()
             .fieldSeparator(',')
             .ofNamedCsvRecord(new InputStreamReader(in, UTF_8))) {

    for (NamedCsvRecord row : csv) {
        // read by column name, not by fragile positional index
        accumulate(toItem(row));
    }
}
```

Reading fields by name (`row.getField("opened_on")`) rather than by position means a reordered or newly-inserted upstream column doesn't silently shift every value by one. Keep the per-field parsing lenient — a blank or malformed optional field should become `null` and a logged warning, not an exception that aborts a three-million-row run.

## Fix 2 — Insert in bounded batches, then forget them

Streaming the input fixes the read side. The write side is still one-insert-at-a-time until you accumulate rows and flush them together. The accumulator is a plain list: append each parsed item; when the list hits the batch size, hand it to the writer and clear it. Don't forget the final partial batch after the loop ends.

```java
private final List<Item> batch = new ArrayList<>();

private void accumulate(Item item) {
    batch.add(item);
    if (batch.size() == batchSize) {   // batchSize = 120, see Fix 3
        ingestDbService.saveBatch(batch);
        batch.clear();
    }
}

// after the stream is exhausted:
if (!batch.isEmpty()) {
    ingestDbService.saveBatch(batch);
}
```

For the write itself you have two good options, and this pipeline uses both depending on the table:

- **Hibernate JDBC batching** for the parent-and-children object graph. Turn it on and Hibernate groups your inserts into batched `PreparedStatement` executions instead of one statement per row. The settings that matter:

  ```properties
  # application.properties
  spring.jpa.properties.hibernate.jdbc.batch_size=720
  spring.jpa.properties.hibernate.order_inserts=true
  spring.jpa.properties.hibernate.order_updates=true
  ```

  `order_inserts` is the quiet hero: it reorders pending inserts so all rows for one table are contiguous, which is what lets Hibernate actually fill a batch instead of flushing a batch of one every time the table changes.

- **A raw multi-row `INSERT` via `JdbcTemplate`** for simple, flat tables — especially when you want database-side conflict handling (more on that in Fix 5). One statement, many value tuples:

  ```java
  jdbc.batchUpdate(
      "INSERT INTO item (external_id, opened_on, status) VALUES (?, ?, ?)",
      batch, batch.size(),
      (ps, item) -> {
          ps.setString(1, item.externalId());
          ps.setObject(2, item.openedOn());
          ps.setString(3, item.status());
      });
  ```

Now the critical half of this step. After each batch you must **flush and clear the persistence context**:

```java
@Transactional
public void saveBatch(List<Item> items) {
    itemRepository.saveAll(items);   // cascades to item_history
    entityManager.flush();           // push this batch's SQL to the DB now
    entityManager.clear();           // detach everything — reset the session
}
```

Without the `clear()`, every entity you have ever saved stays managed for the whole file. Two things go wrong, both quadratic-feeling: memory climbs until the JVM thrashes, and every `flush()` re-runs Hibernate's dirty-check across the entire growing set of managed entities. `clear()` detaches the batch you just wrote so the session size stays flat whether the file has ten thousand rows or ten million.

> **The trap: clear() detaches everything, including things you still hold.**
> After `entityManager.clear()`, every entity from that batch is detached — touching a lazy association on one throws `LazyInitializationException`, and re-saving one inserts a duplicate. Treat a flushed batch as gone. Resolve any IDs or values you need *before* the clear, and never reach back into the previous batch.

## Fix 3 — Size the batch against the 65,535-parameter wall

Why 120? It is not a vibe. PostgreSQL's wire protocol caps a single statement at **65,535 bind parameters** (the count is a 16-bit field). A batched insert binds one parameter per column per row, so the number of rows you can pack into one statement is bounded by:

```
max_rows_per_statement = 65535 / columns_per_row
```

The subtlety is the fan-out. Your batch size is counted in *parent* items, but each parent drags up to 24 history children, and the children are the table with the most rows in flight. So the binding constraint is the child insert, not the parent:

```
history rows per batch   = batchSize × 24
params per history row   ≈ 20 columns
params in one statement  = batchSize × 24 × 20

solve for the limit:
  batchSize × 480 ≤ 65535
  batchSize       ≤ 136
```

136 is the ceiling where the worst-case batch would just fit. You do not pick the ceiling — you pick a number under it with headroom for an extra column appearing upstream or a future bump in history depth. **120** is that number: round, comfortably below 136, and it makes the per-batch arithmetic easy to reason about. Exceed the limit and PostgreSQL rejects the statement outright with a *"too many parameters"* error — and it only triggers on the batches that happen to hit maximum fan-out, so it is exactly the kind of bug that passes every test and fails in production on a Tuesday.

> **Compute it, don't guess it.**
> Pin the limit in code as `floor(65535 / (maxChildrenPerParent × childColumns))` and choose a batch size a margin below it. When someone adds a column or bumps the history window, the formula tells you the new safe ceiling instead of leaving you to rediscover it through an incident.

## Fix 4 — Stop asking the database for IDs one row at a time

Batched inserts fix the insert storm but leave the *other* storm untouched: identifier generation. With a plain sequence strategy, Hibernate fetches the next value from the database before each insert. Batch the inserts all you like — you still pay one round-trip per row just to get a key, and at the child table that is up to 72 million extra calls.

The fix is the **pooled sequence optimizer**, controlled by `allocationSize`. Set it and Hibernate fetches a *block* of IDs in a single call, then hands them out from memory until the block is exhausted. Tune the allocation to your batch shape and ID generation stops being a bottleneck:

```java
@Entity
public class Item {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "item_seq")
    @SequenceGenerator(name = "item_seq", sequenceName = "item_id_seq",
                       allocationSize = 120)   // one fetch per parent batch
    private Long id;
}

@Entity
public class ItemHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "item_history_seq")
    @SequenceGenerator(name = "item_history_seq", sequenceName = "item_history_id_seq",
                       allocationSize = 720)   // matches hibernate.jdbc.batch_size
    private Long id;
}
```

The numbers line up deliberately. The parent allocation (120) equals the batch size, so a 120-item batch draws all its parent IDs from one fetched block. The child allocation (720) matches the Hibernate JDBC batch size, so the children for a batch are covered by a handful of fetches instead of thousands. The per-row `nextval` calls collapse from tens of millions to a few thousand.

> **allocationSize and the database sequence's INCREMENT BY must agree.**
> The pooled optimizer assumes the underlying sequence increments by `allocationSize`. If your sequence is `INCREMENT BY 1` but the entity declares `allocationSize = 120`, Hibernate's in-memory block overlaps IDs the database will hand out again — duplicate-key violations under concurrency. Define the sequence with the matching increment in your Flyway migration: `CREATE SEQUENCE item_id_seq INCREMENT BY 120;`

## Fix 5 — Make the whole thing re-runnable

A multi-minute job that writes millions of rows *will* get interrupted — a deploy, a network blip, a pod eviction. When it does, some batches are committed and the rest are not. You need re-running the same file to be safe, and you almost certainly don't want to build a checkpoint/offset system to get there. Idempotent writes give you the same safety for free.

For the flat parent table, push conflict handling into the database with a natural unique key. The multi-row `INSERT` becomes an upsert that silently skips rows that already landed:

```sql
INSERT INTO item (external_id, scrub_date, opened_on, status)
VALUES (?, ?, ?, ?)
ON CONFLICT (external_id, scrub_date) DO NOTHING
```

For the cascade graph, where `ON CONFLICT` is awkward, fetch the set of keys that already exist for this file *once* at the start of the run and skip any item whose key is already present:

```java
Set<String> alreadyIngested = itemRepository.findExistingKeys(scrubDate);

for (NamedCsvRecord row : csv) {
    String key = row.getField("external_id");
    if (alreadyIngested.contains(key)) continue;   // committed on a prior run
    accumulate(toItem(row));
}
```

Now the recovery procedure is the entire recovery procedure: *run the file again.* Committed batches are recognised and skipped; only the un-ingested remainder is written. No offsets, no resume-token table, no partial-state bookkeeping — just inserts that decline to duplicate themselves.

### A note on triggering and concurrency

Kick the job off behind an endpoint or a queue listener and run it on a background thread (`@Async`) so the trigger returns immediately. Within one file the stream is read serially — one parse loop, one batch in flight at a time — and each batch commits in its own short transaction. Resist the urge to fan a single file across threads sharing one persistence context; `EntityManager` is not thread-safe, and the batch-per-transaction model already keeps each unit of work small and recoverable.

## The payoff

Stack the fixes up and the cost profile changes by orders of magnitude. For a three-million-item file with up to 24 history rows each:

```
                         naive                         batched
  parent inserts          3,000,000  statements   →   25,000  batched statements
  child inserts          ~72,000,000 statements   →   ~100,000 batched statements
  sequence round-trips   ~75,000,000 calls        →   a few thousand block fetches
  persistence context    grows to millions        →   flat, ≤ one batch
  effect on shared DB     CPU pegged, neighbours   →   a brief, polite spike
                          starved
```

The parent table alone drops from three million individual inserts to roughly twenty-five thousand batched ones (`3,000,000 / 120`). The job went from an event the on-call team dreaded to one that finishes before anyone notices — and, just as importantly, stopped stealing CPU from the live services sharing the database.

## Where to take this next

The pipeline above is the high-leverage 90%. Three directions to push it further when you need to:

1. **Reach for `COPY` when ORM overhead stops being worth it.** For pure bulk load with no per-row business logic, PostgreSQL's `COPY` (via the JDBC `CopyManager`) is dramatically faster than any `INSERT` — it bypasses the statement planner entirely. The trade-off is that you give up Hibernate's mapping and conflict handling, so it shines for staging-table loads you reconcile afterwards.
2. **Add `reWriteBatchedInserts=true` to the JDBC URL.** This PostgreSQL driver flag rewrites a batch of single-row inserts into a true multi-row statement on the wire, often a 2–3× win on top of Hibernate batching — and it interacts with the parameter limit from Fix 3, so re-check your math when you enable it.
3. **Make ingestion observable.** Emit a counter of rows ingested, a timer per batch, and a gauge for the current file's progress. When a run is slow you want to see *which* phase — parse, insert, or sequence fetch — rather than guess. Bulk jobs that run unattended need to tell you how they're doing.

---

## Recap

Bulk ingestion melts shared databases not because the data is large but because the naive approach multiplies every row into several network round-trips. The fix is to amortise those round-trips into batches sized by a hard limit you can compute. The recipe:

1. Stream the file from object storage and parse it with a low-allocation streaming reader — never buffer the whole file, read fields by name.
2. Accumulate parsed rows into a fixed-size batch; flush each batch, then `entityManager.flush()` + `clear()` to keep the persistence context flat.
3. Size the batch from PostgreSQL's 65,535-parameter limit: `floor(65535 / (maxChildren × childColumns))`, then back off for headroom — that's where 120 comes from.
4. Set `allocationSize` on your sequence generators (matched to batch and JDBC-batch sizes) so IDs come from in-memory blocks, not a round-trip per row — and match the DB sequence's `INCREMENT BY`.
5. Make writes idempotent — `ON CONFLICT DO NOTHING` or a pre-fetched key set — so an interrupted run is fixed by simply re-running the file.

Do that, and a file that used to be a production incident becomes a background job that finishes quietly and leaves the database's other tenants alone.

---

*Originally published at [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/batch-bulk-ingestion-spring-jpa/). Found this useful? The same batching discipline applies far beyond CSV — queue-draining workers, backfills, and migrations all win from bounded batches, pooled ID allocation, and idempotent writes. The numbers change; the recipe doesn't.*
