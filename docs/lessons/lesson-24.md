# Lesson 24: Add Custom Metrics

## What We're Building

Four application-level counters using Micrometer that track business events:
issues processed, duplicates detected, related issues detected, and emails sent.

---

## Technologies

### Micrometer
Micrometer is a metrics instrumentation library — the "SLF4J for metrics". You write metrics
code against Micrometer's API, and it exports to whatever backend you configure
(Prometheus, Datadog, CloudWatch, etc.) with no code changes.

Key metric types:
- **Counter**: monotonically increasing count. `issues_processed_total++`
- **Gauge**: current value, can go up or down. `active_connections: 42`
- **Timer**: records duration of events. `classification_duration_seconds`
- **DistributionSummary**: records distribution of values. `response_sizes_bytes`

### Naming Conventions
Micrometer uses dot-separated names (`issues.processed`). Prometheus converts them to
underscore-separated names with `_total` suffix for counters (`issues_processed_total`).

Best practices:
- Use lowercase
- Use dots as separators
- Add units as the last segment where applicable (`duration.seconds`, `size.bytes`)
- Counters automatically get `_total` suffix in Prometheus

### Tags (Labels)
Tags add dimensions to metrics. Instead of separate counters for each type:
```java
// BAD: one counter per type
Counter duplicatesCounter = ...;
Counter relatedCounter = ...;

// GOOD: one counter with a tag
Counter.builder("issues.classified")
    .tag("relation_type", relationType.name())
    .register(registry)
    .increment();
```

Tags let you query `issues_classified_total{relation_type="DUPLICATE"}` in Prometheus.

---

## Step-by-Step

### Step 1: Inject `MeterRegistry`

`MeterRegistry` is the central Micrometer registry. Spring Boot auto-configures it.
Inject it wherever you want to record metrics:

```java
@Service
public class MetricsService {

    private final Counter issuesProcessedCounter;
    private final Counter duplicatesDetectedCounter;
    private final Counter relatedDetectedCounter;
    private final Counter emailsSentCounter;

    public MetricsService(MeterRegistry registry) {
        this.issuesProcessedCounter = Counter.builder("issues.processed")
            .description("Total number of GitLab issues processed")
            .register(registry);

        this.duplicatesDetectedCounter = Counter.builder("duplicates.detected")
            .description("Total number of duplicate issue relations detected")
            .register(registry);

        this.relatedDetectedCounter = Counter.builder("related.detected")
            .description("Total number of related issue relations detected")
            .register(registry);

        this.emailsSentCounter = Counter.builder("emails.sent")
            .description("Total number of notification emails sent")
            .register(registry);
    }

    public void recordIssueProcessed() { issuesProcessedCounter.increment(); }
    public void recordDuplicateDetected() { duplicatesDetectedCounter.increment(); }
    public void recordRelatedDetected() { relatedDetectedCounter.increment(); }
    public void recordEmailSent() { emailsSentCounter.increment(); }
}
```

### Step 2: Register metrics in the orchestrator

```java
// In IssueProcessingOrchestrator, after processing:
metricsService.recordIssueProcessed();

for (IssueClassification result : results) {
    if (result.classification().classification() == DUPLICATE) {
        metricsService.recordDuplicateDetected();
    } else if (result.classification().classification() == RELATED) {
        metricsService.recordRelatedDetected();
    }
}
```

```java
// In NotificationService, after sending:
metricsService.recordEmailSent();
```

### Step 3: Add a Timer for LLM classification duration

```java
private final Timer classificationTimer;

public MetricsService(MeterRegistry registry) {
    // ... other metrics ...
    this.classificationTimer = Timer.builder("llm.classification.duration")
        .description("Time taken for LLM to classify an issue pair")
        .register(registry);
}

public <T> T timeClassification(Supplier<T> supplier) {
    return classificationTimer.record(supplier);
}
```

Use it in `LlmClassificationService`:
```java
return metricsService.timeClassification(
    () -> chatClient.prompt().system(SYSTEM_PROMPT).user(userPrompt).call().entity(ClassificationResult.class)
);
```

### Step 4: Verify metrics appear in Actuator

```bash
curl http://localhost:8080/actuator/metrics/issues.processed
```

```json
{
  "name": "issues.processed",
  "measurements": [{ "statistic": "COUNT", "value": 5.0 }]
}
```

### Step 5: Check Prometheus format

```bash
curl http://localhost:8080/actuator/prometheus | grep issues_processed
```

```
# HELP issues_processed_total Total number of GitLab issues processed
# TYPE issues_processed_total counter
issues_processed_total 5.0
```

---

## Practical Tips

**Register metrics in the constructor, not lazily.** If you create a `Counter` inside a method
only when it's first used, the metric won't appear in `/actuator/metrics` until that path
is hit. Register all metrics at construction time so they're visible from startup (with value 0).

**Use `@Timed` for simple method timing.** Instead of injecting `Timer` manually:
```java
@Timed(value = "llm.classification.duration", description = "LLM classification time")
public ClassificationResult classify(...) { ... }
```
Requires `TimedAspect` bean to be registered:
```java
@Bean TimedAspect timedAspect(MeterRegistry registry) { return new TimedAspect(registry); }
```

**Add common tags to all metrics.** Tag all metrics with the application name:
```yaml
management:
  metrics:
    tags:
      application: ai-issue-deduplicator
```
This is invaluable in a Prometheus/Grafana setup where multiple apps report to the same server.

**Learn PromQL basics.** Once metrics are in Prometheus, use PromQL to query them:
```
rate(issues_processed_total[5m])           # issues per second over last 5 min
duplicates_detected_total / issues_processed_total  # duplicate ratio
```
These queries form the basis of Grafana dashboards.
