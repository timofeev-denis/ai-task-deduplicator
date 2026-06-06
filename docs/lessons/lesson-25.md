# Lesson 25: Add OpenTelemetry Tracing

## What We're Building

Distributed tracing across the full pipeline — from webhook receipt through embedding,
similarity search, LLM classification, GitLab update, and email sending — so every
processing run can be inspected as a single trace.

---

## Technologies

### Distributed Tracing Concepts
A **trace** is the complete journey of one request through the system.
A **span** is one unit of work within that journey (e.g., "call GitLab API").
Spans have parent-child relationships, forming a tree.

```
Trace: process issue #42
├── Span: load issue from GitLab          [12ms]
├── Span: generate embedding               [340ms]
├── Span: vector similarity search         [18ms]
├── Span: LLM classify pair (candidate 1)  [1.2s]
├── Span: LLM classify pair (candidate 2)  [0.9s]
├── Span: create GitLab link               [45ms]
└── Span: send email                       [380ms]
```

This tells you: the full pipeline took 2.9s, LLM calls dominate. Actionable insight.

### OpenTelemetry
OpenTelemetry is an open standard for telemetry data (traces, metrics, logs).
It defines APIs, SDKs, and a collector. Your app emits traces in OTLP (OpenTelemetry Protocol),
and any OpenTelemetry-compatible backend (Jaeger, Zipkin, Grafana Tempo) can receive them.

### Micrometer Tracing
Spring Boot 3+ uses Micrometer Tracing as the tracing abstraction (analogous to how
Micrometer is the abstraction for metrics). You write code against Micrometer Tracing's API,
and it delegates to the underlying implementation (OpenTelemetry, Brave/Zipkin).

### Automatic Instrumentation
Spring Boot auto-instruments:
- Incoming HTTP requests (each request becomes a trace)
- `@Scheduled` tasks
- `@Async` method calls
- Spring Data queries
- `RestClient` / `WebClient` outgoing calls

You get all this for free — just adding the dependencies is enough.

---

## Step-by-Step

### Step 1: Add tracing dependencies

```xml
<!-- Micrometer Tracing with OpenTelemetry bridge -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<!-- OpenTelemetry OTLP exporter -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Step 2: Configure tracing in `application.yml`

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # trace 100% of requests (use 0.1 = 10% in production)
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
```

### Step 3: Add Jaeger to Docker Compose for local trace visualization

```yaml
jaeger:
  image: jaegertracing/all-in-one:latest
  container_name: deduplicator-jaeger
  ports:
    - "4318:4318"    # OTLP HTTP receiver
    - "16686:16686"  # Jaeger UI
```

Now open `http://localhost:16686` to see traces.

### Step 4: Add custom spans with `@NewSpan`

Spring auto-instruments HTTP requests and `RestClient` calls, but your business methods
won't automatically get spans. Add `@NewSpan` to key methods:

```java
@NewSpan("generate-embedding")
public float[] generateEmbeddingForIssue(Issue issue) {
    return generateEmbedding(issue.fullText());
}

@NewSpan("llm-classify")
public ClassificationResult classify(ProcessedIssue source, ProcessedIssue candidate) {
    // ...
}

@NewSpan("similarity-search")
public List<SimilarIssue> findSimilar(float[] embedding, Long excludeId) {
    // ...
}
```

### Step 5: Add span tags with `@SpanTag`

```java
@NewSpan("llm-classify")
public ClassificationResult classify(
        ProcessedIssue source,
        @SpanTag("candidate.id") ProcessedIssue candidate) {
    // candidate.getId() is automatically added as a tag to the span
}
```

Or add tags programmatically:
```java
Span.current().tag("issues.processed", String.valueOf(issueIid));
```

### Step 6: Enable log correlation

Add MDC (Mapped Diagnostic Context) support so every log line includes the trace ID:

```yaml
logging:
  pattern:
    console: "%d{HH:mm:ss} [%X{traceId}] %-5level %logger{36} - %msg%n"
```

Now logs look like:
```
14:23:51 [a3b4c5d6e7f8] INFO  IssueProcessingOrchestrator - Processing issue #42...
```

You can paste the trace ID into Jaeger to find the full trace for any log line.

---

## Practical Tips

**Start with automatic instrumentation before adding `@NewSpan`.** Run the app, send a webhook,
then look at Jaeger. You'll already see spans for the HTTP request and database queries.
Add manual spans only where the automatic ones don't provide enough detail.

**Sampling at 1.0 (100%) in development.** In production, use 0.1 or lower — tracing
every request generates large amounts of data. In development, trace everything so you
can always find a specific request.

**Use trace IDs in support.** When something goes wrong in production, the trace ID lets
you reconstruct exactly what happened for that specific request — which services were called,
what data was passed, where the error occurred. This is the core value of distributed tracing.

**Understand propagation.** Trace context (trace ID, span ID) is propagated between services
via HTTP headers (`traceparent`). If you call another service that also has OpenTelemetry,
the trace automatically continues into that service. This is how you get end-to-end traces
in microservices.

**Try Grafana Tempo as an alternative to Jaeger.** Tempo is a modern distributed tracing
backend that integrates with Grafana dashboards. It can correlate traces with metrics and logs
in a single UI — more powerful for production use.
