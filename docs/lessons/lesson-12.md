# Lesson 12: Implement Similarity Search

## What We're Building

A query that finds the top-10 most similar issues to a given embedding using pgvector's
cosine distance operator in a native Spring Data query.

---

## Technologies

### pgvector Similarity Search
pgvector adds three distance operators to PostgreSQL:
- `<=>` — cosine distance (1 - cosine_similarity). Best for text embeddings.
- `<->` — L2 (Euclidean) distance. Best for images.
- `<#>` — negative inner product. Used when vectors are already normalized.

To find the TOP-10 most similar issues:
```sql
SELECT *, 1 - (embedding <=> :queryEmbedding) AS score
FROM processed_issue
WHERE id != :excludeId
ORDER BY embedding <=> :queryEmbedding
LIMIT 10;
```

Note: `<=>` returns **distance** (smaller = more similar). We subtract from 1 to get **similarity** (larger = more similar).

### HNSW Index
The HNSW index (created in Lesson 03) makes this query fast — instead of comparing the query
vector against every row (brute force), it traverses a graph structure to find approximate
nearest neighbors in O(log n) time.

Without the index: O(n) — gets slow with thousands of issues.
With HNSW index: O(log n) — stays fast even with millions of issues.

### Native Queries in Spring Data
For SQL features not expressible in JPQL (like pgvector operators), use `@Query` with
`nativeQuery = true`:

```java
@Query(value = "SELECT * FROM processed_issue WHERE ...", nativeQuery = true)
List<ProcessedIssue> findSimilar(...);
```

---

## Step-by-Step

### Step 1: Add the query method to the repository

```java
public interface ProcessedIssueRepository extends JpaRepository<ProcessedIssue, Long> {

    @Query(value = """
            SELECT *,
                   1 - (embedding <=> CAST(:embedding AS vector)) AS similarity_score
            FROM processed_issue
            WHERE id != :excludeId
              AND embedding IS NOT NULL
            ORDER BY embedding <=> CAST(:embedding AS vector)
            LIMIT :limit
            """,
           nativeQuery = true)
    List<ProcessedIssue> findTopSimilar(
        @Param("embedding") String embedding,
        @Param("excludeId") Long excludeId,
        @Param("limit") int limit
    );
}
```

Note: the embedding is passed as a `String` (the `[0.1, 0.2, ...]` format) because native queries
don't use JPA's `AttributeConverter`. We explicitly `CAST` it to `vector` in SQL.

### Step 2: Create a result DTO

The query returns similarity score which is not a column in the entity.
Use a projection or a simple record:

```java
public record SimilarIssue(ProcessedIssue issue, double similarityScore) {}
```

### Step 3: Create `SimilaritySearchService`

```java
@Service
public class SimilaritySearchService {

    private static final int TOP_K = 10;

    private final ProcessedIssueRepository repository;

    public SimilaritySearchService(ProcessedIssueRepository repository) {
        this.repository = repository;
    }

    public List<SimilarIssue> findSimilar(float[] queryEmbedding, Long excludeIssueId) {
        String vectorString = toVectorString(queryEmbedding);
        List<ProcessedIssue> results = repository.findTopSimilar(
            vectorString, excludeIssueId, TOP_K);

        // Score is not directly available from entity mapping with native query.
        // We compute it here using a second query, or use a @SqlResultSetMapping.
        // For simplicity, use the cosine similarity from a separate calculation.
        return results.stream()
            .map(issue -> new SimilarIssue(issue, computeCosineSimilarity(queryEmbedding, issue.getEmbedding())))
            .sorted(Comparator.comparingDouble(SimilarIssue::similarityScore).reversed())
            .toList();
    }

    private String toVectorString(float[] embedding) {
        StringJoiner joiner = new StringJoiner(",", "[", "]");
        for (float v : embedding) joiner.add(String.valueOf(v));
        return joiner.toString();
    }

    private double computeCosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### Step 4: Verify the HNSW index is being used

Run the query with `EXPLAIN ANALYZE`:
```sql
EXPLAIN ANALYZE
SELECT * FROM processed_issue
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;
```

Look for `Index Scan using idx_processed_issue_embedding` in the output.
If you see `Seq Scan`, the index is not being used (may happen with very few rows — PostgreSQL
prefers seq scan for small tables).

---

## Practical Tips

**Cosine similarity vs cosine distance.** `<=>` returns cosine **distance** = 1 - similarity.
So `embedding <=> query = 0.0` means identical, `1.0` means opposite. When sorting,
`ORDER BY embedding <=> query ASC` gives you most similar first. Don't confuse the two.

**Set a minimum similarity threshold.** Even with TOP-10, some results may be completely
unrelated (low similarity score). Add a filter:
```java
.filter(s -> s.similarityScore() > 0.75)
```
Tune this threshold by testing with real issue data.

**HNSW tuning parameters.** The index has parameters that affect recall vs speed:
- `m` (default 16): connections per node. Higher = better recall, more memory.
- `ef_construction` (default 64): build-time quality. Higher = better index, slower build.

For most use cases, defaults are fine. Only tune if you have many issues and accuracy problems.

**Learn to read `EXPLAIN ANALYZE` output.** It shows:
- Which index is used
- How many rows were scanned
- Actual vs estimated row counts
- Execution time at each step

This is the most important PostgreSQL debugging skill.
