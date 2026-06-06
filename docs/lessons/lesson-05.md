# Lesson 05: Create GitLab HTTP Client

## What We're Building

A declarative HTTP client for the GitLab API using Spring's `@HttpExchange` interface,
capable of fetching issues, creating links, and adding labels.

---

## Technologies

### Spring `@HttpExchange` (Spring 6 / Boot 3+)
Before Spring 6, the standard way to call REST APIs was `RestTemplate` (blocking, verbose)
or `WebClient` (reactive, complex). Spring 6 introduced declarative HTTP clients:
you define an **interface** with annotated methods, and Spring generates the implementation.

This is similar to OpenFeign but built into Spring with no extra dependencies.

```java
@HttpExchange("https://gitlab.com/api/v4")
public interface GitLabClient {

    @GetExchange("/projects/{projectId}/issues/{issueIid}")
    GitLabIssueDto getIssue(@PathVariable long projectId, @PathVariable long issueIid);
}
```

Spring generates a proxy that calls the real HTTP endpoint when you invoke the method.

### RestClient
`RestClient` is the modern blocking HTTP client introduced in Spring Boot 3.2.
It has a fluent builder API similar to `WebClient` but is synchronous.
We use it as the transport layer for `@HttpExchange` proxies.

```java
RestClient restClient = RestClient.builder()
    .baseUrl("https://gitlab.com/api/v4")
    .defaultHeader("PRIVATE-TOKEN", token)
    .build();

HttpServiceProxyFactory factory = HttpServiceProxyFactory
    .builderFor(RestClientAdapter.create(restClient))
    .build();

GitLabClient client = factory.createClient(GitLabClient.class);
```

### GitLab REST API
GitLab exposes a comprehensive REST API at `/api/v4`. Relevant endpoints for this project:
- `GET /projects/:id/issues/:issue_iid` — fetch a single issue
- `POST /projects/:id/issues/:issue_iid/links` — create a link between issues
- `PUT /projects/:id/issues/:issue_iid` — update issue (to add labels)

Authentication: send a `PRIVATE-TOKEN: <your-token>` header with every request.

---

## Step-by-Step

### Step 1: Define GitLab response DTOs

Use Java records — immutable, concise, auto-mapped by Jackson:

```java
public record GitLabIssueDto(
    Long id,
    Long iid,
    Long projectId,
    String title,
    String description,
    String state,
    GitLabAuthorDto author,
    List<String> labels
) {}

public record GitLabAuthorDto(
    Long id,
    String name,
    String username,
    String email
) {}
```

Note: GitLab returns `project_id` (snake_case). Configure Jackson globally to handle this:
```yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
```

### Step 2: Define the `GitLabClient` interface

```java
@HttpExchange
public interface GitLabClient {

    @GetExchange("/projects/{projectId}/issues/{issueIid}")
    GitLabIssueDto getIssue(
        @PathVariable("projectId") long projectId,
        @PathVariable("issueIid") long issueIid
    );

    @PostExchange("/projects/{projectId}/issues/{sourceIssueIid}/links")
    void createIssueLink(
        @PathVariable("projectId") long projectId,
        @PathVariable("sourceIssueIid") long sourceIssueIid,
        @RequestBody CreateIssueLinkRequest request
    );

    @PutExchange("/projects/{projectId}/issues/{issueIid}")
    GitLabIssueDto updateIssue(
        @PathVariable("projectId") long projectId,
        @PathVariable("issueIid") long issueIid,
        @RequestBody UpdateIssueRequest request
    );
}
```

Request body records:
```java
public record CreateIssueLinkRequest(Long targetProjectId, Long targetIssueIid, String linkType) {}
public record UpdateIssueRequest(String addLabels) {}
```

### Step 3: Create a `@Configuration` that registers the client bean

```java
@Configuration
public class GitLabClientConfig {

    @Bean
    public GitLabClient gitLabClient(GitLabProperties properties) {
        RestClient restClient = RestClient.builder()
            .baseUrl(properties.baseUrl() + "/api/v4")
            .defaultHeader("PRIVATE-TOKEN", properties.privateToken())
            .build();

        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(GitLabClient.class);
    }
}
```

(`GitLabProperties` is defined in Lesson 06.)

### Step 4: Handle errors

By default, `RestClient` throws `RestClientException` on 4xx/5xx responses. Add a custom
error handler to provide better error messages:

```java
RestClient restClient = RestClient.builder()
    .baseUrl(...)
    .defaultHeader(...)
    .defaultStatusHandler(HttpStatusCode::isError, (request, response) -> {
        throw new GitLabApiException(
            "GitLab API error: " + response.getStatusCode() +
            " for " + request.getURI()
        );
    })
    .build();
```

---

## Practical Tips

**Test the client in isolation with a `@SpringBootTest` slice.** Use `MockRestServiceServer`
or Wiremock to mock GitLab responses without needing a real GitLab instance. This lets you
test all error cases easily.

**Compare `@HttpExchange` with OpenFeign.** If you've used Feign before, `@HttpExchange` will
feel familiar. The main difference: `@HttpExchange` is built into Spring, requires no extra
dependency, and uses `RestClient` or `WebClient` under the hood.

**Read the GitLab API docs.** Before implementing each client method, read the actual GitLab
API documentation to understand exact request/response shapes, required fields, and error codes.
GitLab's API is well-documented at `docs.gitlab.com/ee/api/`.

**Use `@JsonProperty` for tricky field names.** If a field name doesn't follow snake_case
(e.g., `iid` vs `issue_iid`), use `@JsonProperty("iid")` on the record component to map it explicitly.

**Log HTTP traffic during development.** Add this to `application.yml`:
```yaml
logging:
  level:
    org.springframework.web.client: DEBUG
```
This logs every request and response, invaluable when debugging API integration issues.
