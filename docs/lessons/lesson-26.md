# Lesson 26: Unit Tests

## What We're Building

Unit tests for `SimilaritySearchService`, `LlmClassificationService`, and `GitLabActionsService`
— testing business logic in isolation with mocked dependencies.

---

## Technologies

### JUnit 5 (Jupiter)
The standard Java testing framework. Key annotations:
- `@Test` — marks a test method
- `@BeforeEach` — runs before each test (setup)
- `@DisplayName` — human-readable test name
- `@ParameterizedTest` — run the same test with multiple inputs
- `@ExtendWith` — register test extensions

### Mockito
A mocking library that creates fake implementations of interfaces and classes.
Key operations:
- `mock(Class)` — create a mock
- `when(...).thenReturn(...)` — define mock behavior
- `verify(mock).method(...)` — assert a method was called
- `ArgumentCaptor` — capture arguments passed to a mock

### AssertJ
A fluent assertion library that produces readable test failures:
```java
assertThat(result.classification()).isEqualTo(RelationType.DUPLICATE);
assertThat(candidates).hasSize(3).extracting(RankedCandidate::rank).containsExactly(1, 2, 3);
```

Much more readable than JUnit's `assertEquals(RelationType.DUPLICATE, result.classification())`.

### Unit Test Design Principles
- **One responsibility per test**: each test verifies one specific behavior
- **Arrange-Act-Assert (AAA)**: set up inputs, run the code, assert results
- **Test behavior, not implementation**: test what the code does, not how it does it
- **Fast and isolated**: no database, no network, no filesystem — use mocks

---

## Step-by-Step

### Step 1: Add test dependencies

Spring Boot's test starter includes JUnit 5, Mockito, and AssertJ:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Step 2: Unit test for `CandidateRankingService`

```java
class CandidateRankingServiceTest {

    private final CandidateRankingService service = new CandidateRankingService();

    @Test
    @DisplayName("filters candidates below minimum threshold")
    void filtersBelowThreshold() {
        List<SimilarIssue> input = List.of(
            similarIssue(0.90),
            similarIssue(0.80),
            similarIssue(0.60)  // below 0.75 threshold
        );

        List<RankedCandidate> result = service.rank(input);

        assertThat(result).hasSize(2);
        assertThat(result.get(0).similarityScore()).isEqualTo(0.90);
    }

    @Test
    @DisplayName("assigns sequential ranks starting from 1")
    void assignsCorrectRanks() {
        List<SimilarIssue> input = List.of(similarIssue(0.85), similarIssue(0.80));

        List<RankedCandidate> result = service.rank(input);

        assertThat(result).extracting(RankedCandidate::rank).containsExactly(1, 2);
    }

    @Test
    @DisplayName("returns empty list when no candidates above threshold")
    void returnsEmptyWhenNoCandidates() {
        assertThat(service.rank(List.of())).isEmpty();
    }

    private SimilarIssue similarIssue(double score) {
        return new SimilarIssue(new ProcessedIssue(), score);
    }
}
```

### Step 3: Unit test for `LlmClassificationService`

```java
@ExtendWith(MockitoExtension.class)
class LlmClassificationServiceTest {

    @Mock ChatClient chatClient;
    @Mock ChatClient.CallResponseSpec responseSpec;
    @Mock ChatClient.ChatClientRequestSpec requestSpec;
    @InjectMocks LlmClassificationService service;

    @Test
    @DisplayName("returns DIFFERENT when LLM call fails")
    void returnsDifferentOnFailure() {
        when(chatClient.prompt()).thenReturn(requestSpec);
        when(requestSpec.system(anyString())).thenReturn(requestSpec);
        when(requestSpec.user(anyString())).thenReturn(requestSpec);
        when(requestSpec.call()).thenReturn(responseSpec);
        when(responseSpec.entity(ClassificationResult.class)).thenThrow(new RuntimeException("API unavailable"));

        ClassificationResult result = service.classify(new ProcessedIssue(), new ProcessedIssue());

        assertThat(result.classification()).isEqualTo(RelationType.DIFFERENT);
        assertThat(result.confidence()).isEqualTo(0.0);
    }

    @Test
    @DisplayName("returns classification from LLM response")
    void returnsLlmClassification() {
        var expected = new ClassificationResult(RelationType.DUPLICATE, 0.95, "Same bug");
        when(chatClient.prompt()).thenReturn(requestSpec);
        when(requestSpec.system(anyString())).thenReturn(requestSpec);
        when(requestSpec.user(anyString())).thenReturn(requestSpec);
        when(requestSpec.call()).thenReturn(responseSpec);
        when(responseSpec.entity(ClassificationResult.class)).thenReturn(expected);

        ClassificationResult result = service.classify(new ProcessedIssue(), new ProcessedIssue());

        assertThat(result.classification()).isEqualTo(RelationType.DUPLICATE);
        assertThat(result.confidence()).isEqualTo(0.95);
    }
}
```

### Step 4: Unit test for `GitLabActionsService`

```java
@ExtendWith(MockitoExtension.class)
class GitLabActionsServiceTest {

    @Mock GitLabClient gitLabClient;
    @InjectMocks GitLabActionsService service;

    @Test
    @DisplayName("creates link with correct parameters")
    void createsLink() {
        service.createLink(10L, 42L, 17L, RelationType.DUPLICATE);

        verify(gitLabClient).createIssueLink(
            eq(10L), eq(42L),
            argThat(req -> req.targetIssueIid().equals(17L) && "relates_to".equals(req.linkType()))
        );
    }

    @Test
    @DisplayName("handles 409 Conflict without throwing")
    void handles409Gracefully() {
        doThrow(new GitLabApiException(409, "Conflict"))
            .when(gitLabClient).createIssueLink(anyLong(), anyLong(), any());

        assertThatCode(() -> service.createLink(10L, 42L, 17L, RelationType.DUPLICATE))
            .doesNotThrowAnyException();
    }
}
```

---

## Practical Tips

**Test edge cases, not just the happy path.** For every service, identify:
- What if input is null or empty?
- What if the external dependency throws?
- What if the result is empty?

**Avoid `@SpringBootTest` in unit tests.** `@SpringBootTest` starts the full Spring context —
it's slow (5-10 seconds) and makes tests fragile. For unit tests, instantiate classes directly
or use `@ExtendWith(MockitoExtension.class)` only. Reserve `@SpringBootTest` for integration tests.

**Use `@DisplayName` to document intent.** Test names should read like requirements:
`"filters candidates below minimum threshold"` is much more useful than `"testFilter"`.
When a test fails, you immediately know what requirement broke.

**Run tests with coverage.** In IntelliJ: right-click → "Run tests with coverage".
Aim for 80%+ coverage of your service layer. Coverage alone doesn't guarantee quality,
but it shows you which code paths aren't tested.

**Learn `ArgumentCaptor`.** When you need to assert what arguments were passed to a mock:
```java
ArgumentCaptor<CreateIssueLinkRequest> captor = ArgumentCaptor.forClass(CreateIssueLinkRequest.class);
verify(gitLabClient).createIssueLink(any(), any(), captor.capture());
assertThat(captor.getValue().linkType()).isEqualTo("relates_to");
```
