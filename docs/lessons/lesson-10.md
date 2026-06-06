# Lesson 10: Integrate Embedding Model

## What We're Building

A service that takes an issue's title and description and returns a vector embedding
(an array of 1536 floats) using OpenAI's embedding model via Spring AI.

---

## Technologies

### Embeddings
An embedding is a numerical representation of text in a high-dimensional vector space.
Texts with similar meaning have embeddings that are geometrically close together.

Example:
- "Login button not working" → `[0.023, -0.541, 0.119, ...]` (1536 dimensions)
- "Sign-in fails after update" → `[0.031, -0.538, 0.124, ...]` (very similar vector)
- "Database connection timeout" → `[0.512, 0.033, -0.291, ...]` (very different vector)

Cosine similarity between the first two: ~0.97 (nearly identical)
Cosine similarity between first and third: ~0.21 (very different)

This is how we detect duplicate issues without exact text matching.

### Spring AI
Spring AI is a framework that provides a consistent abstraction over different AI providers
(OpenAI, Azure OpenAI, Anthropic, etc.). You program against a common interface —
switching providers requires only a configuration change, not code changes.

Key interfaces:
- `EmbeddingModel` — generate embeddings for text
- `ChatModel` — generate text responses
- `VectorStore` — store and search embeddings (pgvector implementation available)

### OpenAI `text-embedding-3-small`
OpenAI's efficient embedding model. Key facts:
- Output: 1536-dimensional vector (default)
- Input: up to 8191 tokens (~6000 words)
- Cost: very cheap (~$0.02 per 1M tokens)
- Alternative: `text-embedding-ada-002` (older, same dimensions, slightly higher cost)

---

## Step-by-Step

### Step 1: Add Spring AI OpenAI dependency

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

Add the Spring AI BOM to manage versions:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Step 2: Configure Spring AI in `application.yml`

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY:}
      embedding:
        options:
          model: text-embedding-3-small
```

### Step 3: Create `EmbeddingService`

```java
package com.example.deduplicator.similarity;

@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    public float[] generateEmbedding(String text) {
        EmbeddingResponse response = embeddingModel.embedForResponse(List.of(text));
        List<Double> vector = response.getResults().get(0).getOutput();
        float[] result = new float[vector.size()];
        for (int i = 0; i < vector.size(); i++) {
            result[i] = vector.get(i).floatValue();
        }
        return result;
    }
}
```

### Step 4: Prepare the text input

The quality of similarity search depends on the quality of the embedding input.
Combine title and description into a single text:

```java
public float[] generateEmbeddingForIssue(Issue issue) {
    String text = issue.fullText(); // "title\ndescription"
    return generateEmbedding(text);
}
```

Optionally, weight the title more by repeating it:
```java
String text = issue.title() + "\n" + issue.title() + "\n" + issue.description();
```

This gives the title more influence on the embedding since it's the most discriminating part.

### Step 5: Write a test

```java
@SpringBootTest
class EmbeddingServiceTest {

    @Autowired
    EmbeddingService service;

    @Test
    void generatesEmbeddingOfCorrectDimension() {
        float[] embedding = service.generateEmbedding("Login button not working");
        assertThat(embedding).hasSize(1536);
    }
}
```

Note: this test calls the real OpenAI API. It requires `OPENAI_API_KEY` to be set.
For unit tests, mock `EmbeddingModel`.

---

## Practical Tips

**Understand what you're paying for.** OpenAI charges per token, not per API call.
`text-embedding-3-small` is $0.02 per 1M tokens. A typical issue (50 words) is ~70 tokens.
At 10,000 issues, cost is ~$0.014. Very affordable, but track it.

**Truncate long descriptions.** OpenAI's limit is 8191 tokens. Most issue descriptions are short,
but someone might paste a full stack trace. Truncate to ~2000 tokens to stay safe:
```java
String truncated = text.length() > 8000 ? text.substring(0, 8000) : text;
```

**Cache embeddings — don't regenerate them on every search.** The whole point of TASK-011
(Persist Embeddings) is to store embeddings once and reuse them. Never regenerate an embedding
for an issue that's already in the database.

**Test similarity manually.** Write a simple test that generates embeddings for 3-4 issue titles
and computes cosine similarity between them:
```java
float similarity = cosineSimilarity(embedding1, embedding2);
```
Check that similar issues have similarity > 0.85 and different issues < 0.5.
This builds intuition about the embedding space.
