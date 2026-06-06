# Lesson 09: Publish Domain Event

## What We're Building

The decoupling layer between the webhook and the AI pipeline. The webhook publishes an
`IssueReceivedEvent`, and the pipeline starts asynchronously via Spring Modulith's
Event Publication Registry — with guaranteed delivery even across app restarts.

---

## Technologies

### Domain Events
A domain event is an immutable record of something that happened in the system.
Events decouple producers from consumers — the webhook doesn't know or care who processes
the event or how long it takes.

```java
public record IssueReceivedEvent(long projectId, long issueIid, String action) {}
```

### Spring's `ApplicationEventPublisher`
The standard Spring mechanism for publishing events within the same JVM:

```java
@Component
public class WebhookEventPublisher {
    private final ApplicationEventPublisher publisher;

    public WebhookEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void publish(GitLabWebhookPayload payload) {
        var event = new IssueReceivedEvent(
            payload.objectAttributes().projectId(),
            payload.objectAttributes().iid(),
            payload.objectAttributes().action()
        );
        publisher.publishEvent(event);
    }
}
```

### Spring Modulith Event Publication Registry
The standard Spring `publishEvent` is fire-and-forget: if the listener fails or the app
crashes, the event is lost. Spring Modulith's registry solves this:

1. When `publishEvent` is called within a transaction, the event is **saved to the DB**
   (in `event_publication` table) before the transaction commits.
2. The listener runs asynchronously after the transaction commits.
3. On success, the event is marked `completed`.
4. On failure or restart, incomplete events are available for retry.

This gives you **at-least-once delivery** semantics.

### `@ApplicationModuleListener`
A Spring Modulith annotation for event listeners that combines:
- `@EventListener` — listens for the event type
- `@Transactional` — runs in its own transaction
- `@Async` — runs in a background thread

```java
@ApplicationModuleListener
public void onIssueReceived(IssueReceivedEvent event) {
    // runs asynchronously after the webhook handler returns
}
```

---

## Step-by-Step

### Step 1: Add Spring Modulith Events JPA dependency

```xml
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-api</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-jpa</artifactId>
</dependency>
```

This creates the `event_publication` table via Spring Modulith's own Flyway migration
(or you can create it manually). Verify the table exists after the first startup.

### Step 2: Define the event

```java
package com.example.deduplicator.webhook;

public record IssueReceivedEvent(
    long projectId,
    long issueIid,
    String action
) {}
```

Events should be simple value objects — no business logic, no Spring beans.

### Step 3: Publish the event from the webhook handler

The webhook handler must run inside a transaction for the registry to persist the event:

```java
@Transactional
public void publish(GitLabWebhookPayload payload) {
    var event = new IssueReceivedEvent(
        payload.objectAttributes().projectId(),
        payload.objectAttributes().iid(),
        payload.objectAttributes().action()
    );
    publisher.publishEvent(event);
}
```

The transaction ensures: either the event is saved to the DB AND the webhook returns 200,
or neither happens.

### Step 4: Create the listener in the `issue-processing` module

```java
package com.example.deduplicator.issueprocessing;

@Service
public class IssueProcessingService {

    @ApplicationModuleListener
    public void onIssueReceived(IssueReceivedEvent event) {
        // Full AI pipeline runs here, asynchronously
        // This is invoked AFTER the webhook handler returns
    }
}
```

### Step 5: Enable async processing

In `application.yml`:
```yaml
spring:
  modulith:
    events:
      completion-mode: archive
```

Add `@EnableAsync` to your main application class (or any `@Configuration` class):
```java
@SpringBootApplication
@EnableAsync
public class Application { ... }
```

### Step 6: Configure retry-on-restart

```java
@Bean
ApplicationRunner resubmitIncompleteEvents(IncompleteEventPublications events) {
    return args -> events.resubmitIncompletePublicationsOlderThan(Duration.ZERO);
}
```

This runs at startup and resubmits any events that were not completed in the previous run.

### Step 7: Inspect the `event_publication` table

After sending a webhook, query:
```sql
SELECT * FROM event_publication;
```

You should see a row with `completion_date = null` while the listener is running,
and `completion_date` set after it completes. This gives you full visibility into event processing.

---

## Practical Tips

**Spring Modulith's module boundaries are enforced at test time.** Run:
```java
@Test
void verifyModularStructure() {
    ApplicationModules.of(Application.class).verify();
}
```
This test fails if any module accesses another module's internals. Add it early and keep it green.

**The event is what crosses module boundaries, not the service.** The `webhook` module publishes
the event. The `issue-processing` module listens to it. Neither module imports classes from the other.
This is the fundamental principle of Spring Modulith.

**Understand what `@ApplicationModuleListener` does under the hood.** It is equivalent to:
```java
@EventListener
@Transactional(propagation = Propagation.REQUIRES_NEW)
@Async
```
The `REQUIRES_NEW` means the listener gets its own transaction, separate from the publisher.
If the listener throws, the publisher's transaction is not affected.

**Monitor the async thread pool.** With virtual threads enabled (`spring.threads.virtual.enabled=true`),
async operations use virtual threads automatically. With platform threads, configure the executor:
```yaml
spring:
  task:
    execution:
      pool:
        max-size: 20
```
