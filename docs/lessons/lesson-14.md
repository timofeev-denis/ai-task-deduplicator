# Lesson 14: Integrate Chat Model

## What We're Building

A connection to OpenAI's chat completion API via Spring AI's `ChatClient`,
capable of sending prompts and receiving structured text responses.

---

## Technologies

### Large Language Models (LLMs)
LLMs are neural networks trained on massive text corpora. Given a prompt (input text),
they generate a completion (output text). For our use case, we send two issue descriptions
and ask the model to classify their relationship.

Key concepts:
- **System message**: sets the context and role of the AI ("You are an expert at classifying GitLab issues...")
- **User message**: the actual input ("Here are two issues: ...")
- **Temperature**: controls randomness. `0.0` = deterministic (best for classification tasks).

### Spring AI `ChatClient`
Spring AI's fluent API for interacting with chat models. It abstracts over different providers.
`ChatClient` is the high-level API; `ChatModel` is the low-level API.

```java
String response = chatClient.prompt()
    .system("You are a classification assistant.")
    .user("Classify this: ...")
    .call()
    .content();
```

### `ChatClient.Builder`
Spring AI auto-configures a `ChatClient.Builder` bean. You inject it and build a configured
`ChatClient` with your default settings (model, temperature):

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultOptions(OpenAiChatOptions.builder()
            .model("gpt-4o-mini")
            .temperature(0.0)
            .build())
        .build();
}
```

---

## Step-by-Step

### Step 1: Verify Spring AI dependency is present

From Lesson 10, you already have:
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

This covers both embedding and chat models. No additional dependency needed.

### Step 2: Configure the chat model in `application.yml`

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY:}
      embedding:
        options:
          model: text-embedding-3-small
      chat:
        options:
          model: gpt-4o-mini
          temperature: 0.0
```

`gpt-4o-mini` is the recommended choice: faster, cheaper than `gpt-4o`, and sufficient
for structured classification tasks.

### Step 3: Create a `ChatClientConfig`

```java
package com.example.deduplicator.classification;

@Configuration
public class ChatClientConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder.build();
    }
}
```

Temperature `0.0` is already set in `application.yml`. If you need to override it programmatically:
```java
return builder
    .defaultOptions(OpenAiChatOptions.builder()
        .temperature(0.0)
        .build())
    .build();
```

### Step 4: Test the connection manually

Create a quick smoke-test service:

```java
@Service
public class ChatModelSmokeTest {

    private final ChatClient chatClient;

    public ChatModelSmokeTest(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    public String testConnection() {
        return chatClient.prompt()
            .user("Reply with exactly: OK")
            .call()
            .content();
    }
}
```

Call this from a `CommandLineRunner` or a test and verify you get `"OK"` back.

### Step 5: Understand token limits and costs

`gpt-4o-mini` pricing (approximate):
- Input: $0.15 per 1M tokens
- Output: $0.60 per 1M tokens

A classification prompt (two issue descriptions + instruction) is ~400 tokens.
At 1000 issues per day, cost ≈ $0.06/day. Very manageable.

Context window: 128k tokens — you can send very long issue descriptions without issues.

---

## Practical Tips

**Always set `temperature: 0.0` for classification.** Higher temperatures introduce randomness.
For a task where you need a deterministic `DUPLICATE`, `RELATED`, or `DIFFERENT` answer,
randomness is harmful. Always use `0.0`.

**Understand the difference between `ChatModel` and `ChatClient`.** `ChatModel` is the low-level
API — you build `Prompt` objects manually. `ChatClient` is the high-level fluent API. For most
use cases, prefer `ChatClient`. Use `ChatModel` only if you need fine-grained control.

**Monitor token usage.** Spring AI's response includes usage metadata:
```java
ChatResponse response = chatClient.prompt().user("...").call().chatResponse();
Usage usage = response.getMetadata().getUsage();
log.info("Tokens used: prompt={}, completion={}", usage.getPromptTokens(), usage.getGenerationTokens());
```
Log this in development to understand your actual costs.

**Test with mocked `ChatClient`.** For unit tests, you don't want real API calls.
Spring AI provides a test utility — or simply mock `ChatClient` with Mockito.
Never let unit tests call external APIs.
