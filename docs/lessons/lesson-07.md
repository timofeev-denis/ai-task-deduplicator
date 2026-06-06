# Lesson 07: Load Issue Details

## What We're Building

A service that, given a GitLab project ID and issue IID from a webhook payload,
fetches the full issue details (title, description, author email) from the GitLab API.

---

## Technologies

### Service Layer in Spring
The service layer contains business logic and orchestrates between the web layer (controllers, webhooks)
and the data layer (repositories, HTTP clients). Services are annotated with `@Service`,
which is a specialization of `@Component` — it registers the class as a Spring bean.

Spring's layered architecture:
```
Controller / Webhook handler
        ↓
    Service (@Service)
        ↓
Repository (@Repository) or HTTP Client
```

### Dependency Injection
Spring manages bean lifecycle and wires dependencies automatically. With constructor injection
(the recommended style), dependencies are declared as constructor parameters:

```java
@Service
public class IssueLoaderService {

    private final GitLabClient gitLabClient;

    public IssueLoaderService(GitLabClient gitLabClient) {
        this.gitLabClient = gitLabClient;
    }
}
```

Spring sees one constructor — no `@Autowired` annotation needed. The `GitLabClient` bean
(created in Lesson 05) is injected automatically.

### Domain Model vs DTO
- **DTO (Data Transfer Object)**: a data structure shaped for a specific API response
  (e.g., `GitLabIssueDto` matching GitLab's JSON).
- **Domain model**: your internal representation of business concepts.

It's good practice to convert DTOs into domain objects at the boundary of your system,
so your business logic doesn't depend on the external API's shape.

---

## Step-by-Step

### Step 1: Define an internal `Issue` domain record

```java
package com.example.deduplicator.gitlab;

public record Issue(
    long projectId,
    long issueIid,
    String title,
    String description,
    String authorEmail,
    String authorName
) {
    public String fullText() {
        return title + "\n" + (description != null ? description : "");
    }
}
```

The `fullText()` method will be used in Lesson 10 when generating embeddings.

### Step 2: Create `IssueLoaderService`

```java
@Service
public class IssueLoaderService {

    private final GitLabClient gitLabClient;

    public IssueLoaderService(GitLabClient gitLabClient) {
        this.gitLabClient = gitLabClient;
    }

    public Issue loadIssue(long projectId, long issueIid) {
        GitLabIssueDto dto = gitLabClient.getIssue(projectId, issueIid);
        return toIssue(dto);
    }

    private Issue toIssue(GitLabIssueDto dto) {
        return new Issue(
            dto.projectId(),
            dto.iid(),
            dto.title(),
            dto.description(),
            dto.author().email(),
            dto.author().name()
        );
    }
}
```

### Step 3: Place in the correct module

This service belongs in the `gitlab` module (package `com.example.deduplicator.gitlab`).
It is the only class in this module that should be visible to other modules.

In Spring Modulith, each package is a module. Within a module, only classes in the
**root package** (not subpackages) are part of the public API. Classes in subpackages
(e.g., `com.example.deduplicator.gitlab.internal`) are private to the module.

### Step 4: Write a unit test

```java
class IssueLoaderServiceTest {

    private final GitLabClient gitLabClient = mock(GitLabClient.class);
    private final IssueLoaderService service = new IssueLoaderService(gitLabClient);

    @Test
    void loadsIssueAndMapsFields() {
        var dto = new GitLabIssueDto(1L, 42L, 10L, "Login broken", "Details...",
            "opened", new GitLabAuthorDto(1L, "Alice", "alice", "alice@example.com"), List.of());
        when(gitLabClient.getIssue(10L, 42L)).thenReturn(dto);

        Issue issue = service.loadIssue(10L, 42L);

        assertThat(issue.title()).isEqualTo("Login broken");
        assertThat(issue.authorEmail()).isEqualTo("alice@example.com");
    }
}
```

---

## Practical Tips

**Keep services focused on one responsibility.** `IssueLoaderService` does one thing: fetch
and convert a GitLab issue. Resist the temptation to add embedding generation or similarity
search here — those belong in their own services (Lessons 10, 12).

**Use constructor injection everywhere.** Field injection (`@Autowired` on fields) makes
classes harder to test (you can't instantiate them directly) and hides dependencies.
Constructor injection makes dependencies explicit and testable.

**Map to domain objects at the boundary.** Your `IssueLoaderService` converts `GitLabIssueDto`
to `Issue` immediately after receiving the response. Downstream code should never need to
know what GitLab's API looks like — it works with `Issue`. This makes the code resilient
to GitLab API changes.

**Understand Spring's bean scopes.** By default, Spring beans are singletons — one instance
shared across all requests. This means your services must be **stateless** (no instance
variables that store per-request data). Stateless services are thread-safe by default with
virtual threads.
