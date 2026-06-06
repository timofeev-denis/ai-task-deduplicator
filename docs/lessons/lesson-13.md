# Lesson 13: Create Candidate Ranking

## What We're Building

A ranking step that sorts the top-10 similarity search results by their cosine similarity score
and returns them as a structured list of `RankedCandidate` objects ready for LLM classification.

---

## Technologies

### Java Stream API
Java's functional-style API for processing collections. Relevant operations:
- `stream()` — create a stream from a collection
- `filter()` — keep only elements matching a predicate
- `sorted()` — sort elements
- `map()` — transform each element
- `toList()` — collect results into an immutable list

```java
results.stream()
    .filter(c -> c.similarityScore() > 0.75)
    .sorted(Comparator.comparingDouble(SimilarIssue::similarityScore).reversed())
    .map(this::toRankedCandidate)
    .toList();
```

### Java Records for Domain Objects
Records are ideal for immutable result types. They auto-generate constructor, `equals`,
`hashCode`, and `toString`:

```java
public record RankedCandidate(
    ProcessedIssue issue,
    double similarityScore,
    int rank
) {}
```

### The Role of Ranking
Ranking serves two purposes:
1. **Ordering**: The LLM in Lesson 15 will see candidates in order of likelihood. More similar
   candidates are more likely to be duplicates.
2. **Filtering**: We can cut off candidates below a similarity threshold, reducing unnecessary
   LLM calls (each LLM call costs money and time).

---

## Step-by-Step

### Step 1: Define `RankedCandidate`

```java
package com.example.deduplicator.similarity;

public record RankedCandidate(
    ProcessedIssue issue,
    double similarityScore,
    int rank
) {}
```

### Step 2: Create `CandidateRankingService`

```java
@Service
public class CandidateRankingService {

    private static final double MINIMUM_SIMILARITY_THRESHOLD = 0.75;

    public List<RankedCandidate> rank(List<SimilarIssue> candidates) {
        List<SimilarIssue> filtered = candidates.stream()
            .filter(c -> c.similarityScore() >= MINIMUM_SIMILARITY_THRESHOLD)
            .sorted(Comparator.comparingDouble(SimilarIssue::similarityScore).reversed())
            .toList();

        List<RankedCandidate> ranked = new ArrayList<>();
        for (int i = 0; i < filtered.size(); i++) {
            SimilarIssue candidate = filtered.get(i);
            ranked.add(new RankedCandidate(candidate.issue(), candidate.similarityScore(), i + 1));
        }
        return ranked;
    }
}
```

### Step 3: Integrate into the processing flow

The candidate ranking sits between similarity search and LLM classification:

```
loadIssue()
  → storeIssue() + generateEmbedding()
  → findSimilar()           ← SimilaritySearchService
  → rank()                  ← CandidateRankingService  ← you are here
  → classifyPair() for each ← LlmClassificationService (Lesson 15)
```

### Step 4: Think about the threshold value

`0.75` is a starting point. The right value depends on your data. Ways to tune it:
- Collect 20-30 pairs of known duplicate issues
- Compute their cosine similarity scores
- Set the threshold just below the minimum similarity of confirmed duplicates

If the threshold is too high: you miss duplicates (false negatives).
If the threshold is too low: too many candidates reach the LLM (higher cost, slower).

---

## Practical Tips

**Make thresholds configurable.** Instead of hardcoding `0.75`, put it in `application.yml`:
```yaml
similarity:
  minimum-threshold: 0.75
  top-k: 10
```
And bind it to a `@ConfigurationProperties` record (same pattern as Lesson 06).
This lets you tune without recompiling.

**Log the candidate list.** During development, log each candidate's title and score
after ranking. This lets you visually inspect whether the similarity search is working:
```
Candidate #1: "Login fails after update" (score=0.947)
Candidate #2: "Auth broken on mobile" (score=0.831)
Candidate #3: "Google sign-in error" (score=0.789)
```

**Separate concerns: ranking is not classification.** Ranking decides *which candidates
to consider*. Classification (Lesson 15) decides *what the relationship is*. Keeping them
separate makes each easier to test and tune independently.

**Consider early exit.** If the top candidate has similarity > 0.98, it's almost certainly
a duplicate. You could classify it immediately without ranking the rest. This optimization
can save LLM calls when obvious duplicates are submitted.
