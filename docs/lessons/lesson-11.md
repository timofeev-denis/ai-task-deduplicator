# Lesson 11: Persist Embeddings

## What We're Building

Logic that saves a new issue with its embedding to the database, or updates the embedding
if the issue was already processed (upsert behavior).

---

## Technologies

### Upsert Pattern in JPA
An "upsert" means: insert if not exists, update if exists. JPA doesn't have a native upsert,
but we can implement it cleanly:

1. Try to find the existing entity by a unique key (project ID + issue IID).
2. If found → update fields.
3. If not found → create a new entity.

```java
ProcessedIssue issue = repository
    .findByGitlabProjectIdAndGitlabIssueIid(projectId, issueIid)
    .orElseGet(ProcessedIssue::new);

issue.setTitle(title);
issue.setEmbedding(embedding);
repository.save(issue);
```

`save()` in Spring Data calls `persist()` for new entities (no ID) and `merge()` for existing
ones (with ID). So this pattern works correctly in both cases.

### `@Transactional`
Database operations that must succeed or fail together should be wrapped in a transaction.
`@Transactional` on a method tells Spring to open a transaction before the method and
commit (or rollback on exception) after.

Important: `@Transactional` only works when the method is called through a Spring proxy
(i.e., the caller goes through the Spring bean, not a direct method call within the same class).

---

## Step-by-Step

### Step 1: Create `IssueStorageService`

```java
package com.example.deduplicator.similarity;

@Service
public class IssueStorageService {

    private final ProcessedIssueRepository repository;
    private final EmbeddingService embeddingService;

    public IssueStorageService(ProcessedIssueRepository repository,
                               EmbeddingService embeddingService) {
        this.repository = repository;
        this.embeddingService = embeddingService;
    }

    @Transactional
    public ProcessedIssue storeIssue(Issue issue) {
        float[] embedding = embeddingService.generateEmbeddingForIssue(issue);

        ProcessedIssue entity = repository
            .findByGitlabProjectIdAndGitlabIssueIid(issue.projectId(), issue.issueIid())
            .orElseGet(ProcessedIssue::new);

        entity.setGitlabProjectId(issue.projectId());
        entity.setGitlabIssueIid(issue.issueIid());
        entity.setTitle(issue.title());
        entity.setDescription(issue.description());
        entity.setEmbedding(embedding);

        return repository.save(entity);
    }
}
```

### Step 2: Verify the vector is stored correctly

After calling `storeIssue()`, query the database:

```sql
SELECT id, title, embedding FROM processed_issue LIMIT 1;
```

You should see the title and a vector like `[0.0234,-0.541,0.119,...]` in the `embedding` column.

Also verify the vector dimension:
```sql
SELECT vector_dims(embedding) FROM processed_issue LIMIT 1;
```

Should return `1536`.

### Step 3: Handle the "skip existing" case

When an issue is updated in GitLab, we re-process it. But if the title and description
haven't changed, there's no need to call OpenAI again (saves money):

```java
@Transactional
public ProcessedIssue storeIssue(Issue issue) {
    return repository
        .findByGitlabProjectIdAndGitlabIssueIid(issue.projectId(), issue.issueIid())
        .filter(existing -> !hasContentChanged(existing, issue))
        .orElseGet(() -> createOrUpdate(issue));
}

private boolean hasContentChanged(ProcessedIssue existing, Issue issue) {
    return Objects.equals(existing.getTitle(), issue.title()) &&
           Objects.equals(existing.getDescription(), issue.description());
}
```

### Step 4: Test upsert behavior

```java
@Test
@Transactional
void updatesEmbeddingWhenIssueIsUpdated() {
    // first call — creates
    ProcessedIssue first = service.storeIssue(issue);

    // second call with different content — updates same row
    Issue updated = new Issue(issue.projectId(), issue.issueIid(),
        "Login broken v2", "New description", ...);
    ProcessedIssue second = service.storeIssue(updated);

    assertThat(second.getId()).isEqualTo(first.getId()); // same row
    assertThat(second.getTitle()).isEqualTo("Login broken v2");
}
```

---

## Practical Tips

**Understand `save()` behavior in Spring Data.** `save(entity)` calls `persist()` if `entity.getId() == null`
(new entity) or `merge()` if ID is set (existing entity). The `orElseGet(ProcessedIssue::new)` pattern
leverages this: the new entity has no ID, so it gets persisted; the loaded entity has an ID, so it gets merged.

**Don't call OpenAI inside a transaction unless necessary.** HTTP calls inside transactions keep
the DB connection open longer and increase contention. Consider restructuring to generate the embedding
before opening the transaction, then pass it in:
```java
float[] embedding = embeddingService.generateEmbeddingForIssue(issue); // outside transaction
save(issue, embedding); // @Transactional method
```

**Check the `updated_at` column.** After an update, the `updated_at` timestamp should change.
Verify this in pgAdmin. The `@PreUpdate` lifecycle callback (Lesson 04) handles this.

**Understand flush and dirty checking.** Inside a `@Transactional` method, JPA tracks changes
to loaded entities automatically. You don't always need to call `repository.save()` if you
modified a loaded entity — Hibernate will detect the change and generate an UPDATE on transaction
commit. However, calling `save()` explicitly is clearer and works in all cases.
