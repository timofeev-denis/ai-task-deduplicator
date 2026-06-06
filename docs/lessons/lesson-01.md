# Lesson 01: Create Spring Boot Project

## What We're Building

The foundation of the entire service: a Spring Boot 4 application with all required dependencies declared,
that compiles and starts successfully.

---

## Technologies

### Spring Boot 4
Spring Boot is an opinionated framework that auto-configures Spring components based on what's on the classpath.
Version 4 is built on top of Spring Framework 7 and requires Java 21+. Key ideas:
- **Auto-configuration**: if `spring-data-jpa` is on the classpath and a `DataSource` bean exists, Spring Boot
  automatically creates `EntityManagerFactory`, transaction manager, and repositories — you write zero XML.
- **Starter POMs**: curated sets of dependencies that work well together (e.g., `spring-boot-starter-web` pulls in
  Tomcat, Jackson, Spring MVC with compatible versions).
- **Embedded server**: Tomcat runs inside your JAR — no WAR deployment needed.

### Java 25
Java 25 is an LTS release. Key features relevant to this project:
- **Virtual threads** (stable since Java 21): lightweight threads managed by the JVM, not the OS. Spring Boot 4
  enables them by default for Tomcat — this means each request gets its own virtual thread, giving you
  thread-per-request simplicity without the memory cost.
- **Records**: immutable data classes with auto-generated constructors, `equals`, `hashCode`, `toString`.
  We will use records extensively for DTOs and events.
- **Pattern matching**: cleaner `instanceof` checks and `switch` expressions.

### Maven
Build tool. Manages dependencies, runs tests, packages the JAR. Key files:
- `pom.xml` — project descriptor: dependencies, plugins, Java version.
- `mvnw` — Maven wrapper (bundled with the project so no local Maven install needed).

### Spring Modulith
A library on top of Spring that enforces module boundaries within a single Spring Boot application.
Each package under `com.example.app` becomes a module. Modules communicate only through public APIs
and domain events — Spring Modulith verifies this at test time.

---

## Step-by-Step

### Step 1: Generate the project

Go to [https://start.spring.io](https://start.spring.io) and configure:

| Field | Value |
|-------|-------|
| Project | Maven |
| Language | Java |
| Spring Boot | 4.x (latest) |
| Group | com.example |
| Artifact | ai-issue-deduplicator |
| Java | 25 |

Add dependencies:
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Flyway Migration
- Spring Modulith
- Spring AI (OpenAI)
- Spring Boot Actuator

Download and unzip.

### Step 2: Verify `pom.xml`

Check that `<java.version>25</java.version>` is set and the parent is `spring-boot-starter-parent` 4.x.

The key starters should be present:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
```

### Step 3: Configure `application.yml`

Rename `application.properties` to `application.yml` and add:

```yaml
spring:
  application:
    name: ai-issue-deduplicator
  datasource:
    url: jdbc:postgresql://localhost:5432/deduplicator
    username: deduplicator
    password: secret
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
```

### Step 4: Enable virtual threads

In `application.yml`:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```

This tells Spring Boot to use virtual threads for Tomcat's request handling.

### Step 5: Run the application

```bash
./mvnw spring-boot:run
```

It will fail if PostgreSQL is not running yet — that's expected. We set it up in Lesson 02.
The goal here is that the project **compiles** successfully.

```bash
./mvnw compile
```

This should succeed with `BUILD SUCCESS`.

---

## Practical Tips

**Explore auto-configuration.** Run the app with `--debug` flag and read the `AUTO-CONFIGURATION REPORT`
in the console. You'll see exactly which auto-configurations were applied and why. This demystifies
"Spring magic".

**Read the starter POMs.** In your `.m2/repository` folder, find `spring-boot-starter-web-4.x.pom`
and open it. You'll see exactly what it pulls in. Do this once for each starter you use.

**Use the Spring Boot Reference Docs.** The official docs at `docs.spring.io` are excellent.
Bookmark the "Application Properties" appendix — it lists every `application.yml` property with descriptions.

**Learn the startup sequence.** Spring Boot starts in phases:
1. Create `ApplicationContext`
2. Run auto-configuration
3. Start embedded Tomcat
4. Run `ApplicationRunner`/`CommandLineRunner` beans

Understanding this order helps you debug startup failures.
