---
title: Tracing Requests Across Spring Boot Microservices with a Correlation ID
published: false
description: A practical guide to stamping every log line with a single ID that survives the journey from your API gateway, through every downstream service, and into your error tracker — using SLF4J's MDC, a servlet filter, a Feign interceptor, and Log4J2.
tags: springboot, java, observability, microservices
canonical_url: https://aayushmiglani.github.io/posts/correlation-id-spring-boot/
cover_image:
---

> This article is also available on my blog: [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/correlation-id-spring-boot/) · Code: [github.com/AayushMiglani/correlation-id-spring-boot](https://github.com/AayushMiglani/correlation-id-spring-boot)

## The problem with distributed logs

A modern backend is rarely one application. A typical request enters through an API gateway, fans out to half a dozen downstream services, touches caches and databases, calls a few third-party APIs, and eventually produces a response. Each of those hops writes its own log lines into its own log stream.

When something goes wrong — a 500 response, a duplicated charge, a missing webhook — finding the trail of log lines that belongs to *that one request* is the painful part. You can grep for the user ID and get back a thousand lines from twenty unrelated requests. You can grep for the timestamp and find that twelve other things happened in the same millisecond. Without a single, shared identifier on every line, you are reconstructing the request from circumstantial evidence.

The fix is a **correlation ID**: one identifier, generated at the edge, attached to every log line, propagated to every downstream service, and visible in every Sentry/Bugsnag/Datadog event. With it, debugging a multi-service request is a single search.

> **What you'll build.** A self-contained correlation-ID layer for any Spring Boot service: a filter that mints or accepts an ID per request, a Feign interceptor that forwards it on outbound calls, a Log4J2 configuration that prints it in every log line, and AOP advices that keep it alive across `@Scheduled` and `@Async` boundaries.

## How it works — the 30-second version

The mechanism rests on three Spring Boot concepts working together:

1. **MDC (Mapped Diagnostic Context)** is a thread-local key/value store provided by SLF4J. Whatever you put into MDC at the start of a request, every log statement on that thread can read back — without the caller ever knowing the value is there.
2. **A servlet filter** intercepts every incoming HTTP request. If the request carries a correlation ID header, the filter copies it into MDC; otherwise it mints a fresh UUID. In a `finally` block, it cleans MDC up so the thread is ready for the next request.
3. **A Feign request interceptor** runs on every outbound HTTP call your service makes. It pulls the current correlation ID out of the request context and attaches it as a header on the outbound call, so the downstream service sees the same ID and reuses it.

The diagram below shows the full lifecycle of an ID as a request crosses two services:

```
                ┌──────────────────────────────────────────────────────┐
                │                       Service A                      │
  HTTP request  │                                                      │
  ───────────▶  │  Filter ──▶ DispatcherServlet ──▶ Controller         │
                │     │                              │                 │
                │     │ MDC.put(requestId)           │ Outbound call   │
                │     │ Sentry.setTag(...)           ▼                 │
                │     │                       Feign interceptor:       │
                │     │                       X-Request-ID: <id>       │
                │     │                              │                 │
                └─────┼──────────────────────────────┼─────────────────┘
                      │                              │
                      ▼                              ▼
                Logs tagged              ┌──────────────────────────────┐
                with requestId           │          Service B           │
                                         │                              │
                                         │   Filter reads X-Request-ID  │
                                         │   ──▶ same requestId in MDC  │
                                         └──────────────────────────────┘
```

## Step 1 — Choose Log4J2 over Logback

Spring Boot ships with Logback by default, and Logback works fine for MDC interpolation. But Log4J2 is significantly easier to customise — pattern layouts, async appenders, structured fields, and the Sentry appender all integrate with one or two lines of XML rather than a programmatic configurator. For most production teams, the switch is worth making once and forgetting about.

To make the swap, exclude the default logging starter and add Log4J2:

```gradle
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-log4j2'
}

configurations {
    all {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-logging'
    }
}
```

Or, for Maven users:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

## Step 2 — Write the inbound filter

The filter is the heart of the system. It reads the incoming `X-Request-ID` header (using whichever header name your organisation has standardised on), generates a new UUID if none is present, places the value in MDC under a stable key, and — critically — removes it in a `finally` block so the thread doesn't carry stale state into the next request.

Extending `OncePerRequestFilter` from Spring is the simplest path: it guarantees the filter runs exactly once per request even when the dispatcher forwards internally.

```java
@Component
@Slf4j
@Order(4)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";
    private static final String MDC_KEY = "correlationId";
    private static final String RESPONSE_HEADER = "X-Correlation-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String correlationId = extractOrGenerate(request);
        MDC.put(MDC_KEY, correlationId);
        response.addHeader(RESPONSE_HEADER, correlationId);

        // Optional: attach to error-tracking scope so exceptions carry the ID.
        Sentry.configureScope(scope -> scope.setTag("request_id", correlationId));

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }

    private String extractOrGenerate(HttpServletRequest request) {
        String header = request.getHeader(REQUEST_ID_HEADER);
        if (StringUtils.hasText(header)) {
            return header;
        }
        return UUID.randomUUID().toString().toUpperCase().replace("-", "");
    }

    @Override
    protected boolean shouldNotFilterErrorDispatch() {
        return false;
    }
}
```

A few details worth understanding:

- **Echo the ID on the response.** Adding an `X-Correlation-Id` response header lets clients (and any upstream gateway) record the ID they were assigned without having to read your logs. When a user reports a problem, they can read the ID off the response — often through browser dev tools — and paste it into your search.
- **Always clear MDC in a `finally`.** Servlet containers reuse worker threads. A missing `MDC.remove` means the next request on the same thread will inherit the previous request's ID and your logs will lie to you in the worst possible way.
- **Use `@Order` with intent.** Filter order matters. The correlation-ID filter should run *before* any filter that might log (authentication, request logging, exception translation), so that those filters' log lines also carry the ID.

## Step 3 — Forward the ID on outbound calls

A correlation ID is only useful if the downstream service sees it. With OpenFeign (the most common HTTP client in Spring Boot microservices) this is a one-class job: implement `RequestInterceptor`, register it as a bean, and Feign will apply it to every outbound request.

```java
@Component
public class FeignCorrelationIdInterceptor implements RequestInterceptor {

    private static final String REQUEST_ID_HEADER = "X-Request-ID";
    private static final String MDC_KEY = "correlationId";

    @Override
    public void apply(RequestTemplate template) {
        String correlationId = MDC.get(MDC_KEY);
        if (correlationId != null) {
            template.header(REQUEST_ID_HEADER, correlationId);
        }
    }
}
```

Reading from MDC rather than from `RequestContextHolder` matters: it means the interceptor works the same way whether the outbound call originates from a controller, a scheduled job, an async dispatch, or a deeply nested service method. As long as MDC has the ID, the header gets attached.

> **Using RestTemplate or WebClient?** The same idea applies — register a `ClientHttpRequestInterceptor` (RestTemplate) or an `ExchangeFilterFunction` (WebClient) that reads from MDC and adds the header. The pattern is identical.

## Step 4 — Make the ID visible in every log line

With the filter and interceptor in place, the only thing missing is to render the MDC value in your log output. Log4J2 reads MDC entries via the `%X{key}` pattern token:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Properties>
        <Property name="LOG_PATTERN">
            %d{yyyy-MM-dd HH:mm:ss.SSS} %5p [%t] %-40.40c{1.} : %X{correlationId} - %m%n
        </Property>
    </Properties>
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${LOG_PATTERN}"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```

Save this as `src/main/resources/log4j2-spring.xml` and Spring Boot will pick it up automatically. Any log line emitted on a thread that has `correlationId` in MDC will now include it inline.

A sample log line from a fully wired service looks like:

```
2025-01-14 11:14:02.118  INFO [http-nio-8080-exec-3] c.e.s.controller.OrderController : B3F1A9C24E9B4D6E8F7A12C0D5E61234 - order-received | userId=98421
2025-01-14 11:14:02.245  INFO [http-nio-8081-exec-1] c.e.s.service.PaymentService     : B3F1A9C24E9B4D6E8F7A12C0D5E61234 - payment-authorised | userId=98421 | amount=499
2025-01-14 11:14:02.391  INFO [http-nio-8082-exec-2] c.e.s.service.NotificationService: B3F1A9C24E9B4D6E8F7A12C0D5E61234 - email-queued | userId=98421
```

Three services, three different threads, one ID. Searching for `B3F1A9C24E9B4D6E8F7A12C0D5E61234` in your log aggregator returns the complete picture of the request.

## Step 5 — Don't lose the ID across thread boundaries

Here is where most correlation-ID implementations quietly fall apart. MDC is thread-local. As soon as your code crosses a thread boundary — a `@Scheduled` job runs on a scheduler thread, a `@Async` method gets handed off to an executor, a reactive operator switches schedulers — the MDC value is gone.

There are two complementary fixes; use both.

### 5a. Task decorators for executor pools

Spring's `ThreadPoolTaskExecutor` accepts a `TaskDecorator` that wraps every task submitted to the pool. The decorator captures MDC on the submitting thread and restores it on the worker thread:

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            if (context != null) {
                MDC.setContextMap(context);
            }
            try {
                runnable.run();
            } finally {
                if (previous != null) {
                    MDC.setContextMap(previous);
                } else {
                    MDC.clear();
                }
            }
        };
    }
}

@Configuration
public class AsyncConfig {
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setTaskDecorator(new MdcTaskDecorator());
        executor.setCorePoolSize(8);
        executor.initialize();
        return executor;
    }
}
```

This single decorator covers all `@Async` calls and any task you submit to the pool directly. Now an async dispatch genuinely behaves as a continuation of the originating request, log-wise.

### 5b. AOP advices for @Scheduled jobs

Scheduled jobs are different — they have no originating request, so there is nothing to inherit. They need a *fresh* ID per tick, so each invocation of the job is its own searchable unit:

```java
@Component
@Aspect
public class ScheduledCorrelationAspect {

    private static final String MDC_KEY = "correlationId";

    @Around("@annotation(scheduled)")
    public Object aroundScheduled(ProceedingJoinPoint pjp, Scheduled scheduled) throws Throwable {
        String correlationId = UUID.randomUUID().toString().toUpperCase().replace("-", "");
        MDC.put(MDC_KEY, correlationId);
        try {
            return pjp.proceed();
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

With both pieces in place — task decorator + scheduled aspect — every line of logging in your service carries a correlation ID, regardless of how the work was triggered.

> **The trap most teams hit.** A correlation-ID layer that only covers controllers will look perfect in dev and silently fail in production the moment your service does anything async. If your service has any `@Async` or `@Scheduled` code, treat the task decorator as part of the minimum viable implementation, not a follow-up.

## Searching for an ID

Once IDs are flowing, the operational payoff is immediate. Any log aggregator that supports substring search becomes a distributed-tracing tool.

**AWS CloudWatch Logs Insights:**

```
fields @timestamp, @message, @logStream, @log
| filter @message like 'B3F1A9C24E9B4D6E8F7A12C0D5E61234'
| sort @timestamp asc
| limit 1000
```

**Elasticsearch / Kibana / Loki:**

```
message:"B3F1A9C24E9B4D6E8F7A12C0D5E61234"
```

**Sentry / Bugsnag.** Because the filter binds the ID to the error-tracker's scope as a tag, every captured exception arrives pre-tagged. Filter by `request_id:<id>` to see every related event, or click through from a log line directly to the Sentry issue.

## Where to take this next

The implementation in this article is deliberately minimal — it covers 95% of the operational debugging cases for a small or mid-sized fleet of services. If you want to grow beyond it, three natural next steps are:

1. **Switch to structured (JSON) logging.** Once log lines are JSON, the correlation ID becomes a first-class indexed field rather than a substring match. Aggregator queries get cheaper and faster, and you can filter on combinations like `correlationId=X AND level=ERROR` without brittle regex.
2. **Adopt W3C Trace Context.** A custom `X-Request-ID` header is fine within one team, but if you want to interop with anything OpenTelemetry-aware — Datadog, Honeycomb, Tempo, Jaeger — use the standard `traceparent` header instead. Your filter can read either header and the conversion is mechanical.
3. **Move to full distributed tracing.** Correlation IDs answer "what happened?" Distributed tracing (OpenTelemetry + a backend like Tempo or Honeycomb) also answers "where did it spend its time, and why?" The correlation-ID work here is a stepping stone, not a competitor.

## Recap

A correlation ID is one of the highest-leverage observability primitives you can add to a Spring Boot fleet — a few hundred lines of code that turn cross-service debugging from archaeology into a single search. The full recipe:

1. Use Log4J2 and put `%X{correlationId}` in your pattern layout.
2. Add a `OncePerRequestFilter` that reads `X-Request-ID` or mints a UUID, puts it in MDC, echoes it on the response, and cleans up in a `finally`.
3. Add a `RequestInterceptor` that reads the ID from MDC and forwards it on every outbound Feign call.
4. Install an `MdcTaskDecorator` on every `ThreadPoolTaskExecutor` so `@Async` calls keep the ID.
5. Add an AOP advice that mints a fresh ID for every `@Scheduled` tick.
6. Bind the ID to your error tracker's scope so exceptions arrive pre-tagged.

Ship that, and the next time something breaks in production at 2 a.m., you'll spend the call reading the one log search that matters — not stitching log lines back together by hand.

---

*All the code in this article — filter, interceptor, MDC decorator, scheduled aspect, Log4J2 config — is available as a copy-pasteable package on GitHub: [AayushMiglani/correlation-id-spring-boot](https://github.com/AayushMiglani/correlation-id-spring-boot).*
