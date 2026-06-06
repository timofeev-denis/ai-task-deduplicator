# Lesson 15: Classify Issue Pair

## What We're Building

A service that sends a pair of issues to the LLM and receives a structured classification:
`DUPLICATE`, `RELATED`, or `DIFFERENT`, along with a confidence score.

---

## Technologies

### Prompt Engineering
The art of crafting LLM inputs to get reliable, well-structured outputs. For classification:
- Be explicit about the output format
- Give clear definitions of each category
- Provide examples (few-shot prompting)
- Ask the model to reason step-by-step before giving the final answer

### Spring AI Structured Output
Spring AI can automatically parse LLM responses into Java objects using `BeanOutputConverter`.
Instead of parsing JSON manually, you declare the expected type and Spring AI handles the rest:

```java
chatClient.prompt()
    .user(userPrompt)
    .call()
    .entity(ClassificationResult.class);
```

Spring AI appends a format instruction to your prompt telling the LLM to respond in JSON,
then deserializes the response using Jackson.

### `ClassificationResult` Record

```java
public record ClassificationResult(
    RelationType classification,
    double confidence,
    String reasoning
) {}
```

---

## Step-by-Step

### Step 1: Design the prompt

Good prompt design is the most critical step. The LLM needs clear instructions:

```java
private static final String SYSTEM_PROMPT = """
    You are an expert software QA engineer specializing in issue deduplication.
    Your task is to analyze two GitLab issues and determine their relationship.
    
    Classification categories:
    - DUPLICATE: Both issues describe the same bug or problem. The root cause is identical.
    - RELATED: The issues are different bugs but in the same area or caused by related problems.
    - DIFFERENT: The issues are about completely unrelated bugs or features.
    
    Respond with a JSON object containing:
    - classification: one of DUPLICATE, RELATED, DIFFERENT
    - confidence: a number between 0.0 and 1.0 indicating your certainty
    - reasoning: a brief explanation of your decision (1-2 sentences)
    """;

private String buildUserPrompt(ProcessedIssue source, ProcessedIssue candidate) {
    return """
        Issue A:
        Title: %s
        Description: %s
        
        Issue B:
        Title: %s
        Description: %s
        
        Classify the relationship between Issue A and Issue B.
        """.formatted(
            source.getTitle(), truncate(source.getDescription()),
            candidate.getTitle(), truncate(candidate.getDescription())
        );
}
```

### Step 2: Create `ClassificationResult`

```java
package com.example.deduplicator.classification;

public record ClassificationResult(
    RelationType classification,
    double confidence,
    String reasoning
) {}
```

### Step 3: Create `LlmClassificationService`

```java
@Service
public class LlmClassificationService {

    private static final int MAX_DESCRIPTION_LENGTH = 1000;

    private final ChatClient chatClient;

    public LlmClassificationService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public ClassificationResult classify(ProcessedIssue source, ProcessedIssue candidate) {
        String userPrompt = buildUserPrompt(source, candidate);

        return chatClient.prompt()
            .system(SYSTEM_PROMPT)
            .user(userPrompt)
            .call()
            .entity(ClassificationResult.class);
    }

    private String truncate(String text) {
        if (text == null) return "(no description)";
        return text.length() > MAX_DESCRIPTION_LENGTH
            ? text.substring(0, MAX_DESCRIPTION_LENGTH) + "..."
            : text;
    }
}
```

### Step 4: Handle LLM failures gracefully

The LLM may occasionally return malformed JSON or an unexpected value.
Wrap the call in a try-catch:

```java
public ClassificationResult classify(ProcessedIssue source, ProcessedIssue candidate) {
    try {
        return chatClient.prompt()
            .system(SYSTEM_PROMPT)
            .user(buildUserPrompt(source, candidate))
            .call()
            .entity(ClassificationResult.class);
    } catch (Exception e) {
        log.warn("LLM classification failed for issues {}/{}: {}",
            source.getId(), candidate.getId(), e.getMessage());
        return new ClassificationResult(RelationType.DIFFERENT, 0.0, "Classification failed");
    }
}
```

### Step 5: Test with real examples

Write a test with known pairs and verify the classification is correct:

```java
@SpringBootTest
class LlmClassificationServiceTest {

    @Autowired LlmClassificationService service;

    @Test
    void classifiesObviousDuplicate() {
        // These should be classified as DUPLICATE
        var source = issueWith("Login button broken after v2.1 release", "Clicking login does nothing");
        var candidate = issueWith("Login not working since update", "Login button is unresponsive");

        ClassificationResult result = service.classify(source, candidate);

        assertThat(result.classification()).isEqualTo(RelationType.DUPLICATE);
        assertThat(result.confidence()).isGreaterThan(0.8);
    }
}
```

---

## Practical Tips

**Include `reasoning` in the response.** Even if you don't use it in business logic, the reasoning
field is invaluable for debugging. Log it during development to understand why the LLM made
each decision. It also helps you improve your prompt when classifications are wrong.

**Use few-shot examples for tricky cases.** If RELATED vs DUPLICATE is hard for the model,
add concrete examples directly in the system prompt:
```
Example 1:
Issue A: "Login broken on iOS"
Issue B: "Login broken on Android"
Classification: RELATED (same feature, different platforms)
```

**Validate enum values.** Spring AI deserializes the LLM response with Jackson. If the LLM
returns `"DUPLICATE "` (with a trailing space) or `"duplicate"` (lowercase), Jackson will throw.
Configure Jackson to be case-insensitive for enums:
```java
@JsonProperty("classification")
@JsonFormat(with = JsonFormat.Feature.ACCEPT_CASE_INSENSITIVE_ENUMS)
RelationType classification
```

**Keep the prompt in a resource file for easy editing.** Put the system prompt in
`src/main/resources/prompts/classification-system.st` and load it with Spring AI's `Resource` support.
This avoids Java string escaping and makes prompt iteration faster.
