# Lesson 23: Enable Actuator

## What We're Building

Spring Boot Actuator endpoints for health checking, application info, and metrics exposure —
the foundation for production-grade observability.

---

## Technologies

### Spring Boot Actuator
Actuator adds production-ready features to Spring Boot applications. It exposes HTTP endpoints
that report on the application's internal state. The key endpoints for this project:

| Endpoint | URL | Description |
|----------|-----|-------------|
| Health | `/actuator/health` | App and dependency health |
| Info | `/actuator/info` | App metadata (version, description) |
| Metrics | `/actuator/metrics` | All Micrometer metrics |
| Prometheus | `/actuator/prometheus` | Metrics in Prometheus format |

### Health Indicators
Spring Boot auto-registers health indicators for detected components:
- `DiskSpaceHealthIndicator` — always active
- `DataSourceHealthIndicator` — active when a `DataSource` bean exists (checks DB connectivity)
- `MailHealthIndicator` — active when `JavaMailSender` is configured

The composite result is what `/actuator/health` returns:
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },
    "mail": { "status": "UP" },
    "diskSpace": { "status": "UP" }
  }
}
```

### Security Consideration
By default, most Actuator endpoints are secured or not exposed. We need to explicitly expose
the ones we want. In production, secure them behind authentication or a private network.

---

## Step-by-Step

### Step 1: Verify Actuator dependency is present

From Lesson 01:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Step 2: Configure Actuator in `application.yml`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always
  info:
    env:
      enabled: true
    git:
      mode: full
```

`show-details: always` — show individual component status, not just UP/DOWN.
In production, consider `when-authorized` to hide details from unauthenticated users.

### Step 3: Add application info

In `application.yml`:
```yaml
info:
  app:
    name: AI Issue Deduplicator
    description: Automatically detects duplicate and related GitLab issues
    version: 1.0.0
```

### Step 4: Add the Maven git plugin for git info

```xml
<plugin>
    <groupId>io.github.git-commit-id</groupId>
    <artifactId>git-commit-id-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>revision</goal></goals>
        </execution>
    </executions>
</plugin>
```

After this, `/actuator/info` will include git commit hash, branch, and timestamp.
This is invaluable in production for knowing exactly which code version is running.

### Step 5: Add Prometheus dependency

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Now `/actuator/prometheus` exposes all metrics in Prometheus scrape format.

### Step 6: Create a custom health indicator

Create a health indicator that checks the OpenAI API connection:

```java
@Component
public class OpenAiHealthIndicator implements HealthIndicator {

    private final EmbeddingModel embeddingModel;

    @Override
    public Health health() {
        try {
            embeddingModel.embed("health check");
            return Health.up().build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## Practical Tips

**Hit the endpoints manually after setup:**
```bash
curl http://localhost:8080/actuator/health | jq .
curl http://localhost:8080/actuator/info | jq .
curl http://localhost:8080/actuator/metrics | jq '.names[]'
```

**The `metrics` endpoint lists available metric names.** To get a specific metric:
```bash
curl http://localhost:8080/actuator/metrics/jvm.memory.used | jq .
```

**Never expose sensitive actuator endpoints publicly in production.** Endpoints like
`/actuator/env`, `/actuator/beans`, and `/actuator/mappings` can expose security-sensitive
configuration. Only expose what's needed:
```yaml
management.endpoints.web.exposure.include: health,info,prometheus
```

**Add Actuator to your Docker health check.** In `docker-compose.yml`, for when you
containerize the app:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
  interval: 10s
  timeout: 5s
  retries: 3
```

**Learn what auto-registered metrics are available.** Spring Boot registers metrics for:
JVM memory, GC, thread pools, HTTP request durations, DataSource connections, etc.
Explore them at `/actuator/metrics` — you'll find useful data without writing any code.
