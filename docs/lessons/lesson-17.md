# Lesson 17: Create Issue Links

## What We're Building

A service that creates links between GitLab issues (source → target) using the GitLab API,
for both DUPLICATE and RELATED relationships.

---

## Technologies

### GitLab Issue Links API
GitLab's REST API provides an endpoint to create links between issues:

```
POST /projects/:id/issues/:issue_iid/links
```

Request body:
```json
{
  "target_project_id": 10,
  "target_issue_iid": 42,
  "link_type": "relates_to"
}
```

`link_type` values:
- `"relates_to"` — generic relationship
- `"blocks"` — source blocks target
- `"is_blocked_by"` — source is blocked by target

For our use case, we use `"relates_to"` for both DUPLICATE and RELATED, since GitLab
doesn't have a native "duplicate" link type. The `duplicate-detected` label (Lesson 18)
serves as the duplicate marker.

### Error Handling for Idempotency
If you try to create a link that already exists, GitLab returns `409 Conflict`.
Your service should handle this gracefully — treat it as a success (the link already exists,
which is the desired state).

---

## Step-by-Step

### Step 1: Verify `GitLabClient` has the link creation method

From Lesson 05:
```java
@PostExchange("/projects/{projectId}/issues/{sourceIssueIid}/links")
void createIssueLink(
    @PathVariable("projectId") long projectId,
    @PathVariable("sourceIssueIid") long sourceIssueIid,
    @RequestBody CreateIssueLinkRequest request
);
```

```java
public record CreateIssueLinkRequest(
    Long targetProjectId,
    Long targetIssueIid,
    String linkType
) {}
```

### Step 2: Create `GitLabActionsService`

```java
@Service
public class GitLabActionsService {

    private final GitLabClient gitLabClient;

    public GitLabActionsService(GitLabClient gitLabClient) {
        this.gitLabClient = gitLabClient;
    }

    public void createLink(long projectId, long sourceIssueIid,
                           long targetIssueIid, RelationType relationType) {
        try {
            gitLabClient.createIssueLink(
                projectId,
                sourceIssueIid,
                new CreateIssueLinkRequest(projectId, targetIssueIid, "relates_to")
            );
            log.info("Created {} link: #{} → #{}", relationType, sourceIssueIid, targetIssueIid);
        } catch (GitLabApiException e) {
            if (e.getStatusCode() == 409) {
                log.debug("Link already exists: #{} → #{}", sourceIssueIid, targetIssueIid);
            } else {
                throw e;
            }
        }
    }
}
```

### Step 3: Add error handling to `GitLabClient`

Update the `RestClient` configuration from Lesson 05 to distinguish 409 from other errors:

```java
.defaultStatusHandler(
    status -> status.isError() && status != HttpStatus.CONFLICT,
    (request, response) -> {
        throw new GitLabApiException(response.getStatusCode().value(),
            "GitLab API error for " + request.getURI());
    }
)
```

And handle 409 separately in `GitLabActionsService` by checking the exception's status code.

### Step 4: Create links from the orchestrator

In `IssueProcessingOrchestrator.onIssueReceived()`, after getting classification results:

```java
for (IssueClassification classification : results) {
    if (classification.classification().classification() != RelationType.DIFFERENT) {
        gitLabActionsService.createLink(
            projectId,
            issueIid,
            classification.candidate().issue().getGitlabIssueIid(),
            classification.classification().classification()
        );
    }
}
```

---

## Practical Tips

**Understand GitLab's link direction.** A link from Issue A to Issue B creates a visible
"relates to" entry on both issues in the GitLab UI. You don't need to create the reverse link —
GitLab does this automatically.

**Test with a real GitLab instance.** Create a free account on `gitlab.com` and a test project.
Create two issues manually, then trigger your service. Verify in GitLab UI that the link appears.
This end-to-end verification is more valuable than any unit test for this task.

**Generate a GitLab Personal Access Token.** Go to GitLab → Profile → Access Tokens.
Create a token with `api` scope. Store it as `GITLAB_PRIVATE_TOKEN` in your environment.
Never commit it.

**Consider rate limiting.** GitLab's REST API has rate limits (typically 2000 requests/minute
for authenticated users). For a service processing many issues, add a delay or retry logic
if you hit `429 Too Many Requests`.

**Understand the link types semantically.** Using `"relates_to"` for both DUPLICATE and RELATED
is fine for MVP. If you want richer semantics in GitLab UI, you could use custom labels or
GitLab's "duplicate of" reference syntax in a comment (`/duplicate #123`).
