# Lesson 06: Create GitLab Configuration

## What We're Building

Type-safe, validated configuration for GitLab integration (base URL and private token)
using Spring Boot's `@ConfigurationProperties`.

---

## Technologies

### `@ConfigurationProperties`
The modern Spring Boot way to bind `application.yml` properties to a Java class.
Instead of injecting individual values with `@Value("${gitlab.base-url}")`, you bind
an entire group of related properties to a record or class:

```java
@ConfigurationProperties(prefix = "gitlab")
public record GitLabProperties(String baseUrl, String privateToken) {}
```

Benefits over `@Value`:
- **Type safety**: Spring converts strings to the correct types (Long, Duration, List, etc.)
- **Validation**: add `@Validated` + Bean Validation annotations (`@NotBlank`, `@URL`)
- **IDE support**: IntelliJ and VS Code show autocomplete for your custom properties
- **Testability**: you can instantiate the record directly in tests

### Bean Validation (Jakarta Validation)
A standard API for declaring constraints on Java objects. Annotations like `@NotBlank`, `@NotNull`,
`@Min`, `@Max`, `@URL` are applied to fields. Spring Boot validates `@ConfigurationProperties`
beans at startup if `@Validated` is present — the app refuses to start with a clear error
if required configuration is missing.

---

## Step-by-Step

### Step 1: Add validation dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Step 2: Create the properties record

```java
package com.example.deduplicator.gitlab;

@Validated
@ConfigurationProperties(prefix = "gitlab")
public record GitLabProperties(

    @NotBlank
    String baseUrl,

    @NotBlank
    String privateToken
) {}
```

### Step 3: Register it

Add `@EnableConfigurationProperties` to your main application class or to a `@Configuration` class:

```java
@SpringBootApplication
@EnableConfigurationProperties(GitLabProperties.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Alternatively, annotate the record with `@Configuration` — but the explicit `@EnableConfigurationProperties`
approach is clearer and avoids component scan pollution.

### Step 4: Add values to `application.yml`

```yaml
gitlab:
  base-url: https://gitlab.com
  private-token: ${GITLAB_PRIVATE_TOKEN:}
```

Note: `${GITLAB_PRIVATE_TOKEN:}` means "read from environment variable `GITLAB_PRIVATE_TOKEN`,
default to empty string if not set". The `@NotBlank` constraint will then catch the empty value
at startup — better than a cryptic NullPointerException at runtime.

### Step 5: Use environment variables for secrets

Never commit real tokens to `application.yml`. Use environment variables or a `.env` file:

```bash
# .env (add this to .gitignore!)
GITLAB_PRIVATE_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
```

With Docker Compose, Spring Boot automatically loads `.env` files. Or set them in your shell:
```bash
export GITLAB_PRIVATE_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
./mvnw spring-boot:run
```

### Step 6: Enable IDE autocomplete

Add this dependency to generate a `spring-configuration-metadata.json` file that IDEs use
to provide autocomplete for your custom properties:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

After adding this and rebuilding, IntelliJ will autocomplete `gitlab.base-url` in `application.yml`.

---

## Practical Tips

**Use kebab-case in YAML, camelCase in Java.** Spring Boot's relaxed binding maps
`gitlab.base-url` (YAML) → `baseUrl` (Java field). Both conventions are equivalent.
Stick to kebab-case in YAML — it's the Spring Boot convention.

**Always read secrets from environment variables, never hardcode them.** Use the
`${ENV_VAR:default}` syntax to make the default obvious. For required secrets with no
default, use `${ENV_VAR}` (no colon) — this will throw at startup if the variable isn't set.

**Test your configuration class in isolation:**
```java
@SpringBootTest(properties = {
    "gitlab.base-url=https://gitlab.example.com",
    "gitlab.private-token=test-token"
})
class GitLabPropertiesTest {
    @Autowired GitLabProperties props;

    @Test
    void propertiesAreBound() {
        assertThat(props.baseUrl()).isEqualTo("https://gitlab.example.com");
    }
}
```

**Understand the difference between `@Value` and `@ConfigurationProperties`.**
`@Value` is fine for a single property but becomes unwieldy with many properties.
`@ConfigurationProperties` is the right tool for groups of related settings — and it's
what Spring Boot itself uses internally for all its auto-configuration.
