# Lesson 16: Process Candidate List

## What We're Building

The orchestration layer that takes all ranked candidates for a new issue and runs LLM
classification on each, returning the final list of DUPLICATE and RELATED findings.

---

## Technologies

### Orchestration Service
An orchestration service coordinates multiple other services to implement a complete
business workflow. It doesn't contain business logic itself — it sequences calls.

The pipeline for processing one incoming issue:
```
Issue (loaded from GitLab)
  → storeIssue() → embedding generated and saved
  → findSimilar() → top-10 candidates by vector distance
  → rank()        → candidates above threshold, sorted
  → classify()    → for each candidate: DUPLICATE / RELATED / DIFFERENT
  → filter()      → keep only DUPLICATE and RELATED
  → return list
```

### Sequential vs Parallel Classification
You have ranked candidates and need to classify each against the new issue.
Two approaches:

**Sequential** (simpler):
```java
candidates.stream()
    .map(c -> classify(newIssue, c))
    .toList();
```

**Parallel** (faster):
```java
candidates.parallelStream()
    .map(c -> classify(newIssue, c))
    .toList();
```

With virtual threads (Java 21+), parallel streams are a reasonable choice for I/O-bound
work like LLM API calls. Each classification is an independent HTTP call — there's no reason
to wait for one before starting the next.

---

## Step-by-Step

### Step 1: Define the output type

```java
public record IssueClassification(
    RankedCandidate candidate,
    ClassificationResult classification
) {}
```

### Step 2: Create `IssueProcessingOrchestrator`

```java
@Service
public class IssueProcessingOrchestrator {

    private final IssueLoaderService issueLoaderService;
    private final IssueStorageService issueStorageService;
    private final SimilaritySearchService similaritySearchService;
    private final CandidateRankingService candidateRankingService;
    private final LlmClassificationService classificationService;

    // constructor injection...

    @ApplicationModuleListener
    public void onIssueReceived(IssueReceivedEvent event) {
        List<IssueClassification> results = process(event.projectId(), event.issueIid());
        // trigger GitLab actions and notifications (Lessons 17-22)
    }

    public List<IssueClassification> process(long projectId, long issueIid) {
        // Step 1: Load full issue details from GitLab
        Issue issue = issueLoaderService.loadIssue(projectId, issueIid);

        // Step 2: Store issue with embedding (upsert)
        ProcessedIssue stored = issueStorageService.storeIssue(issue);

        // Step 3: Find similar issues
        List<SimilarIssue> similar = similaritySearchService.findSimilar(
            stored.getEmbedding(), stored.getId());

        // Step 4: Rank candidates
        List<RankedCandidate> candidates = candidateRankingService.rank(similar);

        if (candidates.isEmpty()) {
            return List.of();
        }

        // Step 5: Classify each candidate
        return candidates.stream()
            .map(candidate -> new IssueClassification(
                candidate,
                classificationService.classify(stored, candidate.issue())
            ))
            .filter(c -> c.classification().classification() != RelationType.DIFFERENT)
            .toList();
    }
}
```

### Step 3: Add logging for observability

```java
log.info("Processing issue #{} in project {}. Found {} similar candidates.",
    issueIid, projectId, candidates.size());

log.info("Classification complete. Duplicates: {}, Related: {}",
    results.stream().filter(r -> r.classification().classification() == DUPLICATE).count(),
    results.stream().filter(r -> r.classification().classification() == RELATED).count()
);
```

### Step 4: Handle the empty candidates case

If there are no candidates above the similarity threshold, skip LLM calls entirely
and return early. This is a common case for unique issues and avoids unnecessary API costs.

---

## Practical Tips

**This is the most complex class in the project — test it thoroughly.** Write tests that
mock all dependencies and verify the orchestration logic:
- What happens when there are no similar issues?
- What happens when all candidates are classified as DIFFERENT?
- What happens when the LLM service throws?

**Consider adding `@Transactional` carefully.** The orchestration method is long-running
(multiple API calls). Holding a database transaction for the entire duration is a bad idea —
it keeps a connection checked out from the pool. Structure it so DB writes happen in short
transactions (in the individual services), not in the orchestrator.

**Use structured logging.** Each processing run should have a correlation ID traceable
through all log lines. With OpenTelemetry tracing (Lesson 25), this is automatic.
But even without it, add `projectId` and `issueIid` to every log statement in this flow.

**Separate the listener from the processing logic.** The `@ApplicationModuleListener` method
should be thin — it just calls `process()` and passes results downstream.
Keep `process()` separately testable without Spring context.
