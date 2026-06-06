# Lesson 19: Store Decision History

## What We're Building

Persistence of the classification results to the `issue_relation` table,
creating an auditable history of every detected relationship.

---

## Technologies

### Why Store Decision History?
The `issue_relation` table serves multiple purposes:
1. **Audit trail**: know when and why a link was created
2. **Idempotency**: before creating a GitLab link, check if you already processed this pair
3. **Analytics**: how many duplicates detected per week? Per project?
4. **Debugging**: if a link appears incorrect, you can review the stored confidence score

### `@Transactional` Boundaries
Saving to the DB and creating the GitLab link are separate operations. If we wrapped both
in a single transaction, a GitLab API failure would roll back the DB write — good.
But a DB failure after the link is created in GitLab would leave them inconsistent.

The correct approach: save to DB first, then call GitLab API. The DB write is the source
of truth. If the GitLab call fails, you have the DB record and can retry the GitLab call.

---

## Step-by-Step

### Step 1: Create `IssueRelationService`

```java
@Service
public class IssueRelationService {

    private final IssueRelationRepository repository;
    private final ProcessedIssueRepository issueRepository;

    public IssueRelationService(IssueRelationRepository repository,
                                ProcessedIssueRepository issueRepository) {
        this.repository = repository;
        this.issueRepository = issueRepository;
    }

    @Transactional
    public IssueRelation saveRelation(ProcessedIssue source, ProcessedIssue target,
                                      RelationType relationType, double confidence) {
        // Check if relation already exists (idempotency)
        return repository.findBySourceIssueAndTargetIssue(source, target)
            .orElseGet(() -> {
                IssueRelation relation = new IssueRelation();
                relation.setSourceIssue(source);
                relation.setTargetIssue(target);
                relation.setRelationType(relationType);
                relation.setConfidence(BigDecimal.valueOf(confidence));
                return repository.save(relation);
            });
    }

    public List<IssueRelation> findRelationsForIssue(ProcessedIssue issue) {
        return repository.findBySourceIssue(issue);
    }
}
```

### Step 2: Add the query method to `IssueRelationRepository`

```java
public interface IssueRelationRepository extends JpaRepository<IssueRelation, Long> {

    Optional<IssueRelation> findBySourceIssueAndTargetIssue(
        ProcessedIssue sourceIssue, ProcessedIssue targetIssue);

    List<IssueRelation> findBySourceIssue(ProcessedIssue sourceIssue);
}
```

Spring Data generates the query from the method name. No SQL needed.

### Step 3: Save relations from the orchestrator

In `IssueProcessingOrchestrator`, after classification and before GitLab API calls:

```java
for (IssueClassification result : results) {
    if (result.classification().classification() != RelationType.DIFFERENT) {
        // 1. Save to DB first
        issueRelationService.saveRelation(
            stored,
            result.candidate().issue(),
            result.classification().classification(),
            result.classification().confidence()
        );

        // 2. Then create GitLab link
        gitLabActionsService.createLink(...);
    }
}
```

### Step 4: Query history in pgAdmin

After processing a few issues:
```sql
SELECT
    s.title AS source_title,
    t.title AS target_title,
    r.relation_type,
    r.confidence,
    r.created_at
FROM issue_relation r
JOIN processed_issue s ON s.id = r.source_issue_id
JOIN processed_issue t ON t.id = r.target_issue_id
ORDER BY r.created_at DESC;
```

---

## Practical Tips

**Idempotency is critical here.** The same issue can trigger multiple webhooks (user edits
the description twice). Without the `findBySourceIssueAndTargetIssue` check, you'd create
duplicate `issue_relation` rows. The `UNIQUE (source_issue_id, target_issue_id)` constraint
in the DB (Lesson 03) also guards against this.

**Log confidence scores.** After saving, log:
```
Saved DUPLICATE relation: issue #42 → #17 (confidence=0.94)
```
This is the most useful log statement for debugging wrong classifications later.

**Expose history via an API endpoint (optional enhancement).** Once you have data in
`issue_relation`, it's easy to add:
```
GET /api/issues/{projectId}/{issueIid}/relations
```
This lets users see the history through an API instead of querying the DB directly.

**Understand `BigDecimal` for confidence.** We store confidence as `NUMERIC(4,3)` in the DB
(e.g., `0.912`). Java `double` can represent this, but for DB precision, `BigDecimal` is safer.
`BigDecimal.valueOf(0.912)` is exact; `new BigDecimal(0.912)` may have floating-point noise.
Always use `BigDecimal.valueOf()` when converting from `double`.
