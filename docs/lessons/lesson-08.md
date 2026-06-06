# Lesson 08: Create Webhook Endpoint

## What We're Building

A REST endpoint `POST /api/webhooks/gitlab/issues` that receives GitLab webhook events
for issue creation and updates, validates the GitLab secret token, and returns 200 OK.

---

## Technologies

### Spring MVC `@RestController`
`@RestController` = `@Controller` + `@ResponseBody`. Every method's return value is
automatically serialized to JSON and written to the HTTP response.
`@PostMapping` maps HTTP POST requests to a handler method.

```java
@RestController
@RequestMapping("/api/webhooks/gitlab")
public class GitLabWebhookController {

    @PostMapping("/issues")
    public ResponseEntity<Void> handleIssueEvent(...) { ... }
}
```

### Request Body Deserialization
Spring automatically deserializes the incoming JSON body to a Java object using Jackson:

```java
@PostMapping("/issues")
public ResponseEntity<Void> handle(@RequestBody GitLabWebhookPayload payload) { ... }
```

### `@RequestHeader`
Read a specific HTTP header from the request:

```java
public ResponseEntity<Void> handle(
    @RequestHeader("X-Gitlab-Token") String token,
    @RequestBody GitLabWebhookPayload payload
) { ... }
```

### Webhook Security: Token Validation
GitLab sends a `X-Gitlab-Token` header with every webhook. You configure this token
in GitLab's webhook settings. Your endpoint must verify it matches the expected value.

If the token is missing or wrong → return `403 Forbidden`.
This prevents unauthorized actors from triggering your pipeline by posting to your endpoint.

### `ResponseEntity`
Gives you full control over the HTTP response: status code, headers, body.
`ResponseEntity.ok().build()` → 200 with no body.
`ResponseEntity.status(HttpStatus.FORBIDDEN).build()` → 403 with no body.

---

## Step-by-Step

### Step 1: Define the webhook payload DTO

GitLab sends different event types. For issue events:

```java
public record GitLabWebhookPayload(
    String objectKind,
    GitLabWebhookIssue objectAttributes
) {}

public record GitLabWebhookIssue(
    Long id,
    Long iid,
    Long projectId,
    String title,
    String description,
    String action   // "open" or "update"
) {}
```

### Step 2: Add webhook token to configuration

Add to `GitLabProperties` (or create a separate `WebhookProperties`):

```java
@ConfigurationProperties(prefix = "gitlab")
public record GitLabProperties(
    @NotBlank String baseUrl,
    @NotBlank String privateToken,
    @NotBlank String webhookSecret
) {}
```

In `application.yml`:
```yaml
gitlab:
  base-url: https://gitlab.com
  private-token: ${GITLAB_PRIVATE_TOKEN:}
  webhook-secret: ${GITLAB_WEBHOOK_SECRET:}
```

### Step 3: Create the controller

```java
@RestController
@RequestMapping("/api/webhooks/gitlab")
public class GitLabWebhookController {

    private final GitLabProperties properties;
    private final WebhookEventPublisher eventPublisher;

    public GitLabWebhookController(GitLabProperties properties,
                                   WebhookEventPublisher eventPublisher) {
        this.properties = properties;
        this.eventPublisher = eventPublisher;
    }

    @PostMapping("/issues")
    public ResponseEntity<Void> handleIssueEvent(
            @RequestHeader(value = "X-Gitlab-Token", required = false) String token,
            @RequestBody GitLabWebhookPayload payload) {

        if (!isValidToken(token)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }

        if (!"issue".equals(payload.objectKind())) {
            return ResponseEntity.ok().build();
        }

        String action = payload.objectAttributes().action();
        if ("open".equals(action) || "update".equals(action)) {
            eventPublisher.publish(payload);
        }

        return ResponseEntity.ok().build();
    }

    private boolean isValidToken(String token) {
        return properties.webhookSecret().equals(token);
    }
}
```

### Step 4: Use constant-time comparison for token validation

String `.equals()` is susceptible to timing attacks (it returns early on the first mismatch,
leaking information about where the strings differ). For security tokens, use:

```java
private boolean isValidToken(String token) {
    if (token == null) return false;
    return MessageDigest.isEqual(
        properties.webhookSecret().getBytes(StandardCharsets.UTF_8),
        token.getBytes(StandardCharsets.UTF_8)
    );
}
```

### Step 5: Test the endpoint manually

```bash
# Valid token — should return 200
curl -X POST http://localhost:8080/api/webhooks/gitlab/issues \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: your-secret" \
  -d '{"object_kind":"issue","object_attributes":{"iid":1,"project_id":10,"action":"open","title":"Bug"}}'

# Wrong token — should return 403
curl -X POST http://localhost:8080/api/webhooks/gitlab/issues \
  -H "Content-Type: application/json" \
  -H "X-Gitlab-Token: wrong" \
  -d '{}'
```

---

## Practical Tips

**Always handle missing headers gracefully.** Use `required = false` on `@RequestHeader`
and check for null explicitly. Without `required = false`, Spring returns 400 if the header
is absent — which is correct behavior but hides the fact that it should be a 403.

**Filter on `object_kind` early.** GitLab webhooks can deliver many event types to the same URL.
Check `object_kind == "issue"` before doing any work and return 200 for everything else —
this prevents errors when GitLab sends push events or merge request events to your endpoint.

**GitLab retry behavior.** If your endpoint returns anything other than 2xx, GitLab will retry
the webhook. This is why returning 200 quickly (before processing) is important — and why
the async event pattern in Lesson 09 exists.

**Test security in unit tests, not just manually:**
```java
@Test
void returnsForbiddenWhenTokenIsMissing() {
    mockMvc.perform(post("/api/webhooks/gitlab/issues")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{}"))
        .andExpect(status().isForbidden());
}
```
