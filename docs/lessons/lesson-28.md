# Lesson 28: Configure OpenAPI

## What We're Building

Interactive API documentation via SpringDoc OpenAPI — automatically generated from your
controller code and accessible at `/swagger-ui.html`.

---

## Technologies

### OpenAPI 3.0
A specification format (formerly Swagger) for describing REST APIs in JSON or YAML.
Defines endpoints, request/response schemas, authentication methods, and examples.

SpringDoc generates this automatically from your Spring MVC annotations:
- `@RestController` → path
- `@PostMapping("/api/webhooks/gitlab/issues")` → POST endpoint
- `@RequestBody GitLabWebhookPayload` → request schema
- `@ResponseEntity<Void>` → response schema

### SpringDoc OpenAPI
A library that integrates OpenAPI 3 with Spring Boot. It:
1. Scans your controllers at startup
2. Generates an OpenAPI spec at `/v3/api-docs`
3. Serves Swagger UI at `/swagger-ui.html`

---

## Step-by-Step

### Step 1: Add SpringDoc dependency

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

### Step 2: Configure in `application.yml`

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: alpha
  info:
    title: AI Issue Deduplicator API
    description: GitLab webhook receiver and issue deduplication service
    version: 1.0.0
```

### Step 3: Verify it works

Start the app and open `http://localhost:8080/swagger-ui.html`.
You should see the Swagger UI with your webhook endpoint listed.

### Step 4: Enrich the API documentation with annotations

```java
@RestController
@RequestMapping("/api/webhooks/gitlab")
@Tag(name = "GitLab Webhooks", description = "Receives GitLab webhook events")
public class GitLabWebhookController {

    @Operation(
        summary = "Receive GitLab issue event",
        description = "Processes issue created/updated events and triggers deduplication pipeline"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Event accepted and queued for processing"),
        @ApiResponse(responseCode = "403", description = "Invalid or missing X-Gitlab-Token header")
    })
    @PostMapping("/issues")
    public ResponseEntity<Void> handleIssueEvent(
            @Parameter(description = "GitLab webhook secret token", required = true)
            @RequestHeader(value = "X-Gitlab-Token", required = false) String token,
            @RequestBody GitLabWebhookPayload payload) {
        // ...
    }
}
```

### Step 5: Document the webhook payload

Add schema documentation to your DTOs:

```java
@Schema(description = "GitLab webhook event payload")
public record GitLabWebhookPayload(

    @Schema(description = "Event type — always 'issue' for this endpoint", example = "issue")
    String objectKind,

    @Schema(description = "Issue details")
    GitLabWebhookIssue objectAttributes
) {}
```

### Step 6: Add a global OpenAPI configuration bean

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("AI Issue Deduplicator API")
                .description("Automatically detects duplicate GitLab issues using AI")
                .version("1.0.0")
                .contact(new Contact()
                    .name("Your Team")
                    .email("team@example.com")))
            .addSecurityItem(new SecurityRequirement().addList("GitLab Token"))
            .components(new Components()
                .addSecuritySchemes("GitLab Token",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.APIKEY)
                        .in(SecurityScheme.In.HEADER)
                        .name("X-Gitlab-Token")));
    }
}
```

---

## Practical Tips

**Use Swagger UI to test your endpoint during development.** Instead of `curl`, you can
send requests directly from the browser. Click "Try it out" on any endpoint, fill in the
fields, and execute. Much more convenient for manual testing.

**The generated spec at `/v3/api-docs` is machine-readable.** You can import it into
Postman, Insomnia, or any OpenAPI-compatible tool. You can also use it to generate client
SDKs in any language.

**Exclude internal endpoints from documentation.** Actuator endpoints (`/actuator/*`) and
internal APIs shouldn't appear in the public docs. Configure:
```yaml
springdoc:
  paths-to-exclude: /actuator/**
```

**Keep documentation close to the code.** The best practice is to add `@Operation` and
`@ApiResponse` to every controller method. This ensures docs stay in sync with the code —
unlike separate documentation files that rot.

**Learn the difference between `@Schema` and `@Parameter`.** `@Schema` documents a model
property (in a request/response body). `@Parameter` documents a query parameter, path
variable, or header. Using them correctly makes the Swagger UI much more informative.
