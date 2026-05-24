---
title: "Composing One Screen From a Dozen Services: A Backend-for-Frontend Pattern"
published: false
description: How to build a resilient Backend-for-Frontend that fans out to many services in parallel, assembles a single screen, degrades gracefully on partial failure, and decides what to show each user with a config-driven rules engine.
tags: springboot, java, microservices, architecture
canonical_url: https://aayushmiglani.github.io/posts/backend-for-frontend-aggregation/
cover_image:
---

> This article is also available on my blog: [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/backend-for-frontend-aggregation/)

The home screen of a modern app is not owned by one service — it is a collage of fragments, each owned by a different team. This is how to build the layer that stitches them together: a Backend-for-Frontend that fans out in parallel, survives partial failure, and decides what each user sees with a config-driven rules engine instead of a forest of `if` statements.

## The problem: one screen, a dozen owners

Open the home screen of almost any consumer app and you are looking at a dozen independent decisions rendered side by side. A profile header. A row of quick actions. A promotional banner. A list of recommended items. A "things that need your attention" strip. A bottom sheet that appears once a month. Each of these is, behind the scenes, owned by a different backend service with its own database, its own deploy cadence, and its own on-call rotation.

The naive way to assemble that screen is to let the client do it: the mobile app fires ten requests, one per service, and arranges the answers itself. This falls apart quickly. The client now knows the address of every service, has to orchestrate ten round trips over a flaky mobile network, hard-codes the rules for which sections appear, and ships a new app build every time the layout changes. Worse, every one of those ten teams has to think about mobile-specific concerns — payload shape, versioning, backward compatibility — that have nothing to do with their domain.

The **Backend-for-Frontend** (BFF) pattern moves that assembly to the server. A single service sits between the app and the fleet. The client makes *one* call — "give me the home screen for this user" — and the BFF does the fan-out, the composition, the filtering, and the shaping, returning exactly the payload the screen needs and nothing more.

> **What we'll build.** A BFF that aggregates many services into one screen response. We'll cover the two-phase pipeline (gather context, then build sections), the concurrency model that makes the fan-out fast, the graceful-degradation strategy that keeps a slow service from taking down the whole screen, and a config-driven targeting engine that decides which content each user is eligible to see.

## The shape of the answer

Start at the contract. The client wants a single object whose fields map one-to-one onto the regions of the screen. Nothing in this object hints at how it was assembled — that is the whole point.

```java
public record ScreenResponse(
    ProfileHeader      profile,
    List<QuickAction>  quickActions,
    Banner             topBanner,
    List<Card>         recommended,
    List<Notice>       importantUpdates,
    PromoSheet         promoSheet      // may be null — not every user sees it
) {}
```

Each field is produced independently. The profile header comes from the identity service; the banner is chosen by the targeting engine; the recommendations come from a catalog service filtered by eligibility. Some fields are nullable on purpose: if a section has nothing to show — or the service that powers it is unavailable — the field is simply `null`, and the client renders the screen without that region. That single design decision, *nullable sections*, is what makes graceful degradation possible later.

## Two phases: gather, then build

It is tempting to let each section fetch its own data. Don't. If the profile section calls the identity service and the quick-actions section *also* calls the identity service, you've doubled your load on a downstream and made the screen twice as fragile. Instead, split the work into two distinct phases:

1. **Phase 1 — gather context.** Make every downstream call exactly once, in parallel, and collect the results into a single immutable `ScreenContext` object.
2. **Phase 2 — build sections.** Hand that fully-hydrated context to a set of section builders. Builders do no I/O; they only read the context and shape their slice of the response.

This separation pays for itself many times over. Downstream load is bounded and predictable. Builders become pure functions — trivial to unit-test, because you hand them a context object and assert on the output, with no mocking of HTTP clients. And the two phases have different failure semantics, which we'll exploit.

```
                         ┌─────────────────────── BFF ───────────────────────┐
                         │                                                    │
  GET /screen/{user}     │   Phase 1: gather (parallel I/O)                   │
  ──────────────────▶    │   ┌──────────────────────────────────────────┐    │
                         │   │ identity ─┐                               │    │
                         │   │ catalog  ─┼─▶ join ─▶ ScreenContext        │    │
                         │   │ activity ─┘   (immutable, all fetched once)│    │
                         │   └──────────────────────────────────────────┘    │
                         │                      │                             │
                         │   Phase 2: build (pure, parallel, no I/O)          │
                         │   ┌──────────────────▼───────────────────────┐    │
                         │   │ ProfileBuilder ─┐                         │    │
                         │   │ BannerBuilder  ─┼─▶ assemble ScreenResponse│    │
                         │   │ CardsBuilder   ─┘                         │    │
                         │   └──────────────────────────────────────────┘    │
                         └────────────────────────────────────────────────────┘
                                              │
                                              ▼
                                        one JSON payload
```

## Phase 1: gathering context in parallel

The downstream calls are independent of one another, so running them sequentially is pure waste: ten 80 ms calls back to back is 800 ms of latency the user feels. Run them concurrently and the total is roughly the slowest single call. `CompletableFuture` on a dedicated executor is the cleanest way to express this in Java.

```java
public ScreenContext gatherContext(String userId) {
    CompletableFuture<Profile>  profileF  =
        supplyAsync(() -> identityClient.getProfile(userId), executor);
    CompletableFuture<Activity> activityF =
        supplyAsync(() -> activityClient.getActivity(userId), executor);
    CompletableFuture<List<Item>> catalogF =
        supplyAsync(() -> catalogClient.getItems(userId), executor);
    // ... one future per downstream ...

    try {
        CompletableFuture
            .allOf(profileF, activityF, catalogF /* ... */)
            .get(contextTimeoutSeconds, TimeUnit.SECONDS);   // overall budget
    } catch (TimeoutException e) {
        log.warn("context-timeout | userId={} | budget={}s — returning partial",
                 userId, contextTimeoutSeconds);
    }

    return ScreenContext.builder()
        .profile(resolve(profileF))     // null if this one failed or timed out
        .activity(resolve(activityF))
        .catalog(resolve(catalogF))
        .build();
}
```

Two design choices in that snippet are worth dwelling on.

### An overall time budget, not per-call timeouts only

The `.get(timeout)` on `allOf` puts a single wall-clock budget on the entire gather phase. You almost certainly still want per-client connect/read timeouts at the HTTP layer — but the budget on top guarantees the screen responds in bounded time *even if a downstream's own timeout is misconfigured*. The user gets a screen in, say, three seconds no matter what. A home screen that is 90% complete and instant beats one that is 100% complete and arrives after a thirty-second hang on a dead dependency.

### resolve() turns failure into absence

The helper that reads each future never propagates an exception. It returns the value on success and `null` on any failure or timeout:

```java
private static <T> T resolve(CompletableFuture<T> future) {
    try {
        return future.getNow(null);   // already complete after allOf; null otherwise
    } catch (CompletionException | CancellationException e) {
        return null;                  // a single downstream failing is not fatal
    }
}
```

By the time we build the context, a failed dependency is indistinguishable from one that returned no data: both show up as `null`. The builders in Phase 2 only have to handle one case — "this data is missing" — rather than two.

## Graceful degradation as the default

The thread running through both phases is the same: **a missing fragment must never fail the whole screen.** A screen aggregator is fan-in over a dozen independent systems. If the probability that any one of them is healthy is 99.9%, the probability that *all twelve* are simultaneously healthy is only ~98.8% — so on roughly one request in eighty, something is down. If "something is down" means "blank screen," your app feels broken far more often than any single service is actually broken.

The fix is to make absence a first-class, expected state at every layer:

- **Context:** a failed downstream becomes a `null` field, not a thrown exception.
- **Builders:** given a `null` input, a builder returns an empty or null section rather than blowing up.
- **Response:** a null section is a valid response field; the client omits that region.

> **The failure mode this prevents.** Without explicit degradation, the default behaviour of `CompletableFuture.allOf().join()` is to throw if *any* future failed. Ship that, and a single slow or down dependency turns into a 500 for the entire home screen. The whole architecture hinges on deciding, deliberately, that partial data is success.

## Phase 2: the section builder, as a strategy

Twelve sections means twelve chunks of "take the context, produce this part of the response." That is the **Strategy pattern** almost by definition: one interface, many interchangeable implementations, each pluggable and independently testable.

```java
public interface SectionBuilder<T> {
    T build(ScreenContext context);
}
```

Each implementation owns exactly one region of the screen and reads only the parts of the context it needs. Here's the recommendations builder — note that it copes with a missing catalog without any special-casing from the caller:

```java
@Component
public class RecommendedBuilder implements SectionBuilder<List<Card>> {

    @Override
    public List<Card> build(ScreenContext ctx) {
        if (ctx.catalog() == null) {
            return List.of();           // catalog service was down — show nothing
        }
        return ctx.catalog().items().stream()
            .filter(item -> isEligible(ctx, item))
            .map(Card::from)
            .toList();
    }
}
```

Because a builder is a pure function of the context, its test is a one-liner of arrange-act-assert: construct a context, call `build`, assert on the section. No HTTP, no database, no clock, no mocks. When a product manager asks "why did this user see that banner," you can reproduce the decision deterministically from a captured context.

### Running and isolating the builders

The aggregator runs the builders — also in parallel, since they're independent — and wraps each in a boundary so that a bug in one builder can't take out the others:

```java
private <T> CompletableFuture<T> runBuilder(
        SectionBuilder<T> builder, ScreenContext ctx, String name) {
    return CompletableFuture
        .supplyAsync(() -> builder.build(ctx), executor)
        .exceptionally(ex -> {
            log.error("section-build-failed | section={} | userId={}",
                      name, ctx.userId(), ex);
            metrics.increment("section_build_failure", "section", name);
            return null;                // this section is absent; screen still ships
        });
}
```

The `.exceptionally(...)` is the second half of the degradation contract — the Phase 1 `resolve()` handles "the data never arrived"; this handles "the builder threw while shaping it." Either way the outcome is the same: a null section and a screen that still renders.

## The concurrency plumbing you can't skip

Fanning work out across a thread pool quietly breaks two things that worked fine on a single request thread: **request context** and **error attribution**. Both are easy to fix and painful to debug if you don't.

Anything stored in a thread-local — the correlation ID in SLF4J's MDC, the authenticated user, the app version negotiated for this request — lives on the request thread and does *not* follow work onto an executor thread. The moment a builder runs on a pool thread, your logs lose the correlation ID and your traces fragment. Spring's `TaskDecorator` closes the gap by capturing the context on the submitting thread and restoring it on the worker:

```java
public class ContextPropagatingDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable task) {
        Map<String, String> captured = MDC.getCopyOfContextMap();   // submitter thread
        return () -> {
            if (captured != null) MDC.setContextMap(captured);       // worker thread
            try { task.run(); }
            finally { MDC.clear(); }
        };
    }
}

@Bean
public ThreadPoolTaskExecutor screenExecutor(TaskExecutionProperties props) {
    var executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(props.getPool().getCoreSize());
    executor.setMaxPoolSize(props.getPool().getMaxSize());
    executor.setQueueCapacity(props.getPool().getQueueCapacity());
    executor.setTaskDecorator(new ContextPropagatingDecorator());
    executor.initialize();
    return executor;
}
```

The same decorator should also forward the correlation ID onto outbound HTTP calls so the whole fan-out shares one ID across services. I wrote that piece up separately — [tracing requests across services with a correlation ID](https://aayushmiglani.github.io/posts/correlation-id-spring-boot/) — and a screen aggregator is exactly the kind of fan-out that makes it indispensable.

> **Size the pool on purpose.** A screen aggregator's threads spend almost all their time blocked on downstream I/O, not burning CPU. That argues for a pool considerably larger than your core count — and a bounded queue, so that under overload you shed load fast instead of piling up latency. Use a dedicated executor for this work; don't share the framework's default pool with unrelated tasks that have different blocking profiles.

## Deciding what each user sees: a config-driven rules engine

Now the harder half. A builder doesn't just shape data — it has to decide *whether* a section appears and *which variant*. Which banner? Which products are this user eligible for? In what order? The instinct is to encode this in code:

```java
// the slope you do not want to start down
if (user.segment().equals("NEW") && user.daysSinceSignup() < 30
        && appVersion.atLeast("4.2") && !blocklist.contains(user.id())) {
    return Banner.WELCOME_OFFER;
} else if (...) {
    ...
}
```

Every targeting change is now a code change, a deploy, and a release. Marketing wants to launch a promotion on Friday; engineering is the bottleneck. Within a quarter this branch is forty cases deep and nobody can say what any given user will see. The way out is to treat targeting as **data, not code**: a rules engine that reads campaign definitions from a table and evaluates them against the request context.

### The campaign model

A campaign is a small, declarative record. Four fields carry the whole idea:

- **Level** — the targeting granularity: an individual `USER`, a `GROUP` they belong to, or a broad `SEGMENT`. Levels are ranked, so a campaign aimed at a specific user beats a campaign aimed at their whole segment.
- **Value** — the entity the campaign targets at that level (a user id, a group id, a segment name).
- **Filter conditions** — extra predicates that must *all* hold for the campaign to apply (e.g. "account age under 30 days"). AND semantics.
- **Rules** — an ordered, priority-keyed map of what to show: priority → list of candidate content. Lower priority number wins; within a tier, the first eligible candidate is chosen.

```java
public record Campaign(
    String              identifier,
    CampaignLevel       level,          // USER > GROUP > SEGMENT
    String              value,
    Instant             startsAt,
    Instant             endsAt,
    List<String>        filterConditions,
    Map<Integer, List<String>> rules    // {1: ["A"], 2: ["B:variant", "C"]}
) {}
```

### Storing rules without a join table

The `rules` and `filterConditions` fields are variable-shaped — a map of lists, a list of strings. Modelling them as fully normalized child tables would be a lot of JPA ceremony for data that is only ever read and written as a whole. A pragmatic alternative is to serialize them to JSON and store them in a single text/JSONB column, with a JPA `AttributeConverter` doing the translation transparently:

```java
@Converter
public class RulesConverter
        implements AttributeConverter<Map<Integer, List<String>>, String> {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<Integer, List<String>> rules) {
        try { return MAPPER.writeValueAsString(rules); }
        catch (JsonProcessingException e) {
            throw new TargetingException("rules-serialize-failed", e);
        }
    }

    @Override
    public Map<Integer, List<String>> convertToEntityAttribute(String json) {
        try {
            return MAPPER.readValue(json,
                new TypeReference<LinkedHashMap<Integer, List<String>>>() {});
        } catch (IOException e) {
            throw new TargetingException("rules-deserialize-failed", e);
        }
    }
}
```

Two details matter here. Deserialize into a `LinkedHashMap` so insertion order survives the round trip — priority ordering is meaningful and you don't want it scrambled. And wrap the Jackson exceptions in a domain-specific exception that carries a greppable reason code, rather than letting a raw `JsonProcessingException` leak out of the persistence layer.

### Evaluation: precedence, conditions, eligibility

With campaigns in the table, choosing content for a user is three steps:

1. **Find candidate campaigns** that are currently active (now between `startsAt` and `endsAt`) and target this user at *some* level — by their id, their group, or their segment.
2. **Order by precedence** — most specific level first (`USER` over `GROUP` over `SEGMENT`), then most recently created — and pick the first whose filter conditions all pass.
3. **Walk the rules** in priority order; within each priority tier, return the first candidate the user is actually *eligible* for.

```java
public List<Content> resolve(ScreenContext ctx) {
    return activeCampaignsFor(ctx)                       // step 1
        .stream()
        .sorted(BY_LEVEL_THEN_RECENCY)                   // step 2
        .filter(c -> conditionsMatch(c.filterConditions(), ctx))
        .findFirst()
        .map(c -> evaluateRules(c, ctx))                 // step 3
        .orElse(List.of());
}

private boolean conditionsMatch(List<String> conditions, ScreenContext ctx) {
    for (String condition : conditions) {               // AND — short-circuit
        if (!evaluate(condition, ctx)) return false;
    }
    return true;
}

private List<Content> evaluateRules(Campaign campaign, ScreenContext ctx) {
    return campaign.rules().entrySet().stream()
        .sorted(Map.Entry.comparingByKey())             // priority order
        .flatMap(tier -> tier.getValue().stream()
            .map(ContentRef::parse)
            .filter(ref -> eligibilityChecker.isEligible(ctx, ref))
            .findFirst()                                 // first eligible in tier
            .stream())
        .toList();
}
```

The last piece, the `EligibilityChecker`, is where the rules engine meets reality. A campaign can *offer* a piece of content, but the user must still qualify for it given the live context — they haven't already adopted it, their app version supports it, their account is in the right state. Keeping eligibility in one checker, separate from the campaign-selection logic, means the same rule ("is this user eligible for X") is enforced identically whether X is being chosen by the targeting engine, listed in a catalog, or checked by a standalone endpoint. One source of truth for "can this user see this," many callers.

> **Why this structure earns its keep.** Levels give you precedence (target a VIP user directly without disturbing their segment's campaign). Filter conditions give you slicing (same campaign, narrowed to new accounts). Priority-keyed rules give you fallback (offer A; if they already have it, offer B). And because it's all rows in a table, a targeting change is a config update — reviewable, auditable, and reversible without a deploy.

## Observability: know what the screen actually showed

A screen this dynamic is opaque unless you measure it. Two families of metrics make it legible. First, *build health* per section — invoked, succeeded, failed — so a downstream degradation shows up as a spike in `section_build_failure{section="recommended"}` long before it shows up in support tickets. Second, *what the user saw*: increment a counter when a section is populated and another when it comes back empty, tagged by the dimensions you target on (segment, group, app version).

```java
if (section == null || section.isEmpty()) {
    metrics.increment("screen_section_empty", "section", name, "segment", ctx.segment());
} else {
    metrics.increment("screen_section_shown", "section", name, "segment", ctx.segment());
}
```

The empty-rate per section is the single most useful number this service emits. A section that is empty for 80% of users is either mis-targeted or pointing at an unhealthy dependency — and you'd never see either from request-level success rates alone, because returning an empty section *is* a successful request.

## Where to take this next

1. **Cache the context, not the response.** Several fragments change slowly (a profile, group membership). Caching those context pieces with a short TTL cuts downstream load dramatically, while still recomputing the cheap, fast-changing parts on every request. Cache the inputs to the decision, not the decision itself.
2. **Make the time budget adaptive.** A fixed three-second budget is blunt. Track the p99 of each downstream and trip a circuit breaker on the ones that are misbehaving, so a known-bad dependency is skipped immediately rather than waited on every single request.
3. **Promote rule conditions to a small expression language.** Hand-named conditions ("account age under 30 days") are fine until you have fifty of them. A tiny, sandboxed expression grammar over the context lets non-engineers compose conditions without a new deploy for each one — at the cost of building a safe evaluator, which is real work. Don't reach for it until the hard-coded condition list genuinely hurts.

## Recap

A Backend-for-Frontend turns a chaotic client-side fan-out into one clean call, and the discipline that makes it robust is small but non-negotiable:

1. Give the client a flat response whose fields map onto screen regions, with nullable sections.
2. Split into two phases: gather all context once in parallel, then build sections from it as pure functions.
3. Put a wall-clock budget on the gather phase and turn every downstream failure into a `null`, not an exception.
4. Model sections as a `SectionBuilder` strategy, run them in parallel, and isolate each with `.exceptionally()`.
5. Propagate request context across the thread pool with a `TaskDecorator` so logs and traces survive the fan-out.
6. Drive "what to show whom" from a config-backed rules engine — levels for precedence, conditions for slicing, priority rules for fallback — not from branching code.
7. Measure what each section actually rendered; the per-section empty rate is your earliest warning signal.

Build it this way and the screen stays fast when a dependency is slow, stays up when a dependency is down, and changes what it shows without waiting on a release. The complexity doesn't disappear — it moves to where it belongs: behind one interface, expressed as data, observable from the outside.

---

*Related: if you build a fan-out service like this, you'll want every log line across the fan-out to share one ID — see [Tracing Requests Across Spring Boot Microservices with a Correlation ID](https://aayushmiglani.github.io/posts/correlation-id-spring-boot/).*
