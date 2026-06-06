# Lesson 04: Create JPA Entities

## What We're Building

Two JPA entity classes (`ProcessedIssue`, `IssueRelation`) and their Spring Data repositories,
mapped to the tables created in Lesson 03.

---

## Technologies

### JPA (Jakarta Persistence API)
JPA is a standard API for mapping Java objects to database tables (ORM — Object-Relational Mapping).
Key annotations:
- `@Entity` — marks a class as a database entity
- `@Table` — specifies the table name
- `@Id` — marks the primary key field
- `@GeneratedValue` — auto-increment strategy
- `@Column` — customizes column mapping (name, nullable, length)
- `@ManyToOne`, `@OneToMany` — relationship mappings

JPA is a **specification**. Hibernate is the **implementation** Spring Boot uses by default.

### Spring Data JPA
A layer on top of JPA that generates repository implementations automatically.
You define an interface extending `JpaRepository<T, ID>` and Spring generates the implementation
at startup — no SQL, no boilerplate.

```java
public interface ProcessedIssueRepository extends JpaRepository<ProcessedIssue, Long> {
    Optional<ProcessedIssue> findByGitlabProjectIdAndGitlabIssueIid(Long projectId, Long issueIid);
}
```

Spring Data reads the method name and generates the query. No implementation needed.

### pgvector + JPA
The `vector` column type from pgvector is not a standard SQL type, so JPA doesn't know how to map it.
We need a custom `AttributeConverter` that converts between `float[]` (Java) and the `vector` string
representation that PostgreSQL's JDBC driver understands.

---

## Step-by-Step

### Step 1: Create the `ProcessedIssue` entity

```java
package com.example.deduplicator.shared.persistence;

@Entity
@Table(name = "processed_issue")
public class ProcessedIssue {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "gitlab_project_id", nullable = false)
    private Long gitlabProjectId;

    @Column(name = "gitlab_issue_iid", nullable = false)
    private Long gitlabIssueIid;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "text")
    private String description;

    @Column(columnDefinition = "vector(1536)")
    @Convert(converter = VectorConverter.class)
    private float[] embedding;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void prePersist() {
        createdAt = OffsetDateTime.now();
        updatedAt = OffsetDateTime.now();
    }

    @PreUpdate
    void preUpdate() {
        updatedAt = OffsetDateTime.now();
    }

    // getters and setters (or use Lombok @Data/@Getter/@Setter)
}
```

### Step 2: Create the `VectorConverter`

```java
package com.example.deduplicator.shared.persistence;

@Converter
public class VectorConverter implements AttributeConverter<float[], String> {

    @Override
    public String convertToDatabaseColumn(float[] attribute) {
        if (attribute == null) return null;
        StringJoiner joiner = new StringJoiner(",", "[", "]");
        for (float v : attribute) joiner.add(String.valueOf(v));
        return joiner.toString();
    }

    @Override
    public float[] convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        String[] parts = dbData.substring(1, dbData.length() - 1).split(",");
        float[] result = new float[parts.length];
        for (int i = 0; i < parts.length; i++) result[i] = Float.parseFloat(parts[i].trim());
        return result;
    }
}
```

### Step 3: Create the `IssueRelation` entity

```java
@Entity
@Table(name = "issue_relation")
public class IssueRelation {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "source_issue_id", nullable = false)
    private ProcessedIssue sourceIssue;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "target_issue_id", nullable = false)
    private ProcessedIssue targetIssue;

    @Enumerated(EnumType.STRING)
    @Column(name = "relation_type", nullable = false)
    private RelationType relationType;

    @Column(nullable = false, precision = 4, scale = 3)
    private BigDecimal confidence;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @PrePersist
    void prePersist() {
        createdAt = OffsetDateTime.now();
    }
}
```

```java
public enum RelationType {
    DUPLICATE, RELATED
}
```

### Step 4: Create repositories

```java
public interface ProcessedIssueRepository extends JpaRepository<ProcessedIssue, Long> {
    Optional<ProcessedIssue> findByGitlabProjectIdAndGitlabIssueIid(
        Long gitlabProjectId, Long gitlabIssueIid);
}
```

```java
public interface IssueRelationRepository extends JpaRepository<IssueRelation, Long> {
}
```

### Step 5: Enable SQL logging during development

In `application.yml`:
```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

This shows every generated query with bound parameters. Disable in production.

---

## Practical Tips

**Use `FetchType.LAZY` for all associations.** The default (`EAGER`) fetches related entities
immediately in every query — even when you don't need them. `LAZY` loads them only when accessed.
This prevents N+1 query problems.

**Use `@PrePersist` and `@PreUpdate` for timestamps.** This is simpler than `@CreatedDate`/`@LastModifiedDate`
(which requires `@EnableJpaAuditing`) and works without extra configuration.

**Validate the schema with `ddl-auto: validate`.** Set this in `application.yml`:
```yaml
spring.jpa.hibernate.ddl-auto: validate
```
On startup, Hibernate will verify that every entity field maps to a real column.
If your entity doesn't match the migration, you get a clear error at startup.

**Learn what Hibernate generates.** With SQL logging enabled, write a test that saves and loads
an entity, then read the generated SQL. You'll quickly understand what JPA does under the hood.

**Avoid `@Data` from Lombok on entities.** Lombok's `@Data` generates `equals`/`hashCode` based on
all fields — including the `id`. This breaks JPA's entity identity semantics (two entities with
`id = null` would be considered equal). Use `@Getter`/`@Setter` instead.
