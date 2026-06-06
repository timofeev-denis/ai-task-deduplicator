# Lesson 18: Add Duplicate Label

## What We're Building

Logic that adds the `duplicate-detected` label to a GitLab issue when it's classified as a DUPLICATE.

---

## Technologies

### GitLab Labels API
Labels in GitLab are tags that can be applied to issues. They're visible in the issue list
and can be used for filtering. The API to update an issue's labels:

```
PUT /projects/:id/issues/:issue_iid
```

Request body to add a label (without removing existing ones):
```json
{
  "add_labels": "duplicate-detected"
}
```

Note: use `add_labels` (not `labels`). The `labels` field **replaces** all existing labels.
`add_labels` **appends** to existing labels — this is what you want.

### Idempotency
GitLab silently ignores adding a label that's already on the issue. So calling this API
multiple times for the same issue is safe — it's idempotent.

---

## Step-by-Step

### Step 1: Verify `GitLabClient` has the update method

From Lesson 05:
```java
@PutExchange("/projects/{projectId}/issues/{issueIid}")
GitLabIssueDto updateIssue(
    @PathVariable("projectId") long projectId,
    @PathVariable("issueIid") long issueIid,
    @RequestBody UpdateIssueRequest request
);
```

```java
public record UpdateIssueRequest(String addLabels) {}
```

### Step 2: Add `addDuplicateLabel` to `GitLabActionsService`

```java
private static final String DUPLICATE_LABEL = "duplicate-detected";

public void addDuplicateLabel(long projectId, long issueIid) {
    try {
        gitLabClient.updateIssue(projectId, issueIid, new UpdateIssueRequest(DUPLICATE_LABEL));
        log.info("Added label '{}' to issue #{} in project {}",
            DUPLICATE_LABEL, issueIid, projectId);
    } catch (GitLabApiException e) {
        log.warn("Failed to add label to issue #{}: {}", issueIid, e.getMessage());
    }
}
```

Note: we catch and log the exception rather than rethrowing. Failing to add a label is not
critical — the link was already created. Don't let a label failure block email notifications.

### Step 3: Call it from the orchestrator

Only add the label when at least one DUPLICATE is found:

```java
boolean hasDuplicate = results.stream()
    .anyMatch(r -> r.classification().classification() == RelationType.DUPLICATE);

if (hasDuplicate) {
    gitLabActionsService.addDuplicateLabel(projectId, issueIid);
}
```

### Step 4: Create the label in GitLab (one-time setup)

Before the label can be applied, it must exist in the GitLab project. Create it manually:
1. Open your GitLab project
2. Issues → Labels → New label
3. Name: `duplicate-detected`, Color: choose any (red or orange works well visually)

Or create it via the API (can be part of a setup script):
```bash
curl --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_PRIVATE_TOKEN" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/labels" \
  --data "name=duplicate-detected&color=%23FF0000"
```

---

## Practical Tips

**Label creation is a one-time setup concern.** Consider creating the label automatically
at application startup using a `CommandLineRunner`:
```java
@Bean
CommandLineRunner ensureLabelsExist(GitLabClient client, GitLabProperties props) {
    return args -> client.createLabelIfNotExists(props.projectId(), "duplicate-detected", "#FF0000");
}
```
This makes the app self-configuring — no manual GitLab setup needed.

**Add the label name to `application.yml`.** The label name `"duplicate-detected"` is a
configuration concern, not a code concern. Put it in config so it can be changed without recompiling:
```yaml
gitlab:
  duplicate-label: duplicate-detected
```

**Understand `add_labels` vs `labels` in GitLab API.** This is a common mistake.
Using `labels` replaces ALL labels on the issue — you'd erase any labels the QA team
already added. Always use `add_labels` when you want to append.

**Test the label in GitLab UI.** After running the service against a real issue,
check GitLab to confirm the label appears. The label should be visible in the issue
details sidebar and in the issue list view.
