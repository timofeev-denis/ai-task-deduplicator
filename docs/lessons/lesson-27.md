# Lesson 27: Integration Tests

## What We're Building

Integration tests that test the full stack with a real database (via Testcontainers),
a real webhook endpoint (via MockMvc), and a real email server (via GreenMail).

---

## Technologies

### `@SpringBootTest`
Starts the complete Spring application context for testing. Unlike unit tests (which test
one class in isolation), integration tests verify that all components work together.

Modes:
- `MOCK` (default): starts context with mocked Servlet environment, uses MockMvc
- `RANDOM_PORT`: starts real embedded Tomcat on a random port
- `DEFINED_PORT`: uses port from `application.yml`

### Testcontainers
A Java library that starts real Docker containers for testing. Instead of an in-memory H2
database (which doesn't support pgvector), Testcontainers starts the actual PostgreSQL
with pgvector extension in a Docker container for the test, then removes it afterward.

```java
@Testcontainers
class MyIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("pgvector/pgvector:pg17");
}
```

### MockMvc
Tests Spring MVC controllers without starting a real HTTP server. Sends requests through
the full Spring MVC stack (filters, interceptors, controllers) but entirely in-process.

```java
mockMvc.perform(post("/api/webhooks/gitlab/issues")
        .header("X-Gitlab-Token", "secret")
        .contentType(MediaType.APPLICATION_JSON)
        .content(payload))
    .andExpect(status().isOk());
```

### GreenMail
An in-memory SMTP/IMAP server for testing. Replaces Mailpit in tests — captures emails
sent by `JavaMailSender` and lets you assert on them programmatically:

```java
MimeMessage[] receivedMessages = greenMail.getReceivedMessages();
assertThat(receivedMessages[0].getSubject()).contains("дублирующиеся задачи");
```

---

## Step-by-Step

### Step 1: Add Testcontainers and GreenMail dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.icegreen</groupId>
    <artifactId>greenmail-spring</artifactId>
    <version>2.1.0</version>
    <scope>test</scope>
</dependency>
```

### Step 2: Create a base integration test class

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@Testcontainers
@ActiveProfiles("test")
abstract class BaseIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("pgvector/pgvector:pg17");

    @Autowired protected MockMvc mockMvc;
    @Autowired protected ObjectMapper objectMapper;
}
```

`@ServiceConnection` — Spring Boot 3.1+ annotation that automatically configures
`spring.datasource.*` from the container's connection details. No manual property overrides needed.

### Step 3: Create `application-test.yml`

```yaml
spring:
  ai:
    openai:
      api-key: test-key
      # Use a mock for AI services in tests
  mail:
    host: localhost
    port: 3025  # GreenMail default test SMTP port

gitlab:
  base-url: http://localhost:8090  # WireMock or MockServer
  private-token: test-token
  webhook-secret: test-secret
```

### Step 4: Webhook integration test

```java
class WebhookIntegrationTest extends BaseIntegrationTest {

    @Test
    @DisplayName("returns 200 for valid webhook with correct token")
    void acceptsValidWebhook() throws Exception {
        String payload = """
            {
              "object_kind": "issue",
              "object_attributes": {
                "iid": 1,
                "project_id": 10,
                "action": "open",
                "title": "Login broken"
              }
            }
            """;

        mockMvc.perform(post("/api/webhooks/gitlab/issues")
                .header("X-Gitlab-Token", "test-secret")
                .contentType(MediaType.APPLICATION_JSON)
                .content(payload))
            .andExpect(status().isOk());
    }

    @Test
    @DisplayName("returns 403 when token is missing")
    void rejectsMissingToken() throws Exception {
        mockMvc.perform(post("/api/webhooks/gitlab/issues")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("returns 403 when token is wrong")
    void rejectsWrongToken() throws Exception {
        mockMvc.perform(post("/api/webhooks/gitlab/issues")
                .header("X-Gitlab-Token", "wrong-secret")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isForbidden());
    }
}
```

### Step 5: Email integration test with GreenMail

```java
@SpringBootTest
class EmailNotificationIntegrationTest extends BaseIntegrationTest {

    @Autowired GreenMail greenMail;
    @Autowired DuplicateNotificationService notificationService;

    @BeforeEach
    void setUp() {
        greenMail.start();
    }

    @AfterEach
    void tearDown() {
        greenMail.stop();
    }

    @Test
    @DisplayName("sends duplicate notification with correct subject")
    void sendsDuplicateNotification() throws Exception {
        // Arrange
        Issue sourceIssue = new Issue(10L, 42L, "Login broken", "Details", "alice@test.com", "Alice");
        List<IssueClassification> classifications = /* build test data */;

        // Act
        notificationService.sendDuplicateNotification(sourceIssue, classifications, "alice@test.com");

        // Assert
        MimeMessage[] messages = greenMail.getReceivedMessages();
        assertThat(messages).hasSize(1);
        assertThat(messages[0].getSubject()).contains("#42");
        assertThat(messages[0].getAllRecipients()[0].toString()).isEqualTo("alice@test.com");
    }
}
```

### Step 6: Verify Spring Modulith structure

```java
class ModularStructureTest {

    @Test
    void verifyModularStructure() {
        ApplicationModules.of(Application.class).verify();
    }
}
```

This test will fail if any module accesses another module's internals.
Add it early and keep it green — it's your architectural guardrail.

---

## Practical Tips

**Testcontainers is slow — use `@Container static`.** A static container is shared across
all tests in the class. A non-static container restarts for every test method — very slow.
Always declare Testcontainers containers as `static`.

**Mock external APIs with WireMock.** For GitLab API and OpenAI API calls, use WireMock
to serve fake responses. Don't call real external APIs in integration tests:
```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <scope>test</scope>
</dependency>
```

**Keep the CI pipeline fast.** Integration tests are inherently slower than unit tests.
Separate them into a different Maven lifecycle phase (`failsafe` plugin runs `*IT.java` files)
and run them separately from unit tests in CI.

**Use `@Transactional` on test methods for DB cleanup.** Instead of manually deleting data,
annotate test methods with `@Transactional` — the test DB transaction is rolled back after
each test, leaving the DB in a clean state. Works automatically with Testcontainers.
