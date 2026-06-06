# Lesson 03: Create Database Schema

## What We're Building

Two database tables — `processed_issue` (with a pgvector `embedding` column) and `issue_relation` —
created through Flyway migrations.

---

## Technologies

### Flyway
A database migration tool. Instead of running SQL scripts manually, you place versioned SQL files
in `src/main/resources/db/migration/`. Flyway runs them in order on startup and tracks which ones
have already been applied in a `flyway_schema_history` table.

**Why migrations instead of `ddl-auto: create`?**
Spring's `ddl-auto: create` recreates the schema on every restart — you lose all data.
`ddl-auto: validate` (what we use) only checks that the schema matches your entities.
Flyway gives you a controlled, auditable history of schema changes.

Naming convention: `V{version}__{description}.sql`
- `V1__create_initial_schema.sql`
- `V2__add_confidence_index.sql`

The double underscore (`__`) separates version from description.

### pgvector Extension

Before using the `vector` type, the extension must be enabled in PostgreSQL:
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

This only needs to happen once per database. We put it in the first migration.

### HNSW Index

HNSW (Hierarchical Navigable Small World) is an approximate nearest-neighbor index algorithm.
It trades a small amount of recall accuracy for dramatically faster search at scale.

```sql
CREATE INDEX ON processed_issue USING hnsw (embedding vector_cosine_ops);
```

- `vector_cosine_ops` — use cosine distance (`<=>`) for similarity
- `vector_l2_ops` — use L2 (Euclidean) distance (`<->`)
- `vector_ip_ops` — use inner product (`<#>`)

For text embeddings, cosine distance is the standard choice.

---

## Step-by-Step

### Step 1: Create the migrations directory

```
src/main/resources/db/migration/
```

Flyway looks here by default. This is already configured by the `spring-boot-starter-data-jpa`
auto-configuration when Flyway is on the classpath.

### Step 2: Create `V1__create_initial_schema.sql`

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Table for storing processed GitLab issues with their embeddings
CREATE TABLE processed_issue (
    id              BIGSERIAL PRIMARY KEY,
    gitlab_project_id BIGINT       NOT NULL,
    gitlab_issue_iid  BIGINT       NOT NULL,
    title           TEXT          NOT NULL,
    description     TEXT,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT uq_project_issue UNIQUE (gitlab_project_id, gitlab_issue_iid)
);

-- HNSW index for fast cosine similarity search
CREATE INDEX idx_processed_issue_embedding
    ON processed_issue
    USING hnsw (embedding vector_cosine_ops);

-- Table for storing detected issue relations
CREATE TABLE issue_relation (
    id              BIGSERIAL PRIMARY KEY,
    source_issue_id BIGINT        NOT NULL REFERENCES processed_issue(id),
    target_issue_id BIGINT        NOT NULL REFERENCES processed_issue(id),
    relation_type   VARCHAR(20)   NOT NULL CHECK (relation_type IN ('DUPLICATE', 'RELATED')),
    confidence      NUMERIC(4,3)  NOT NULL CHECK (confidence BETWEEN 0 AND 1),
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),

    CONSTRAINT uq_relation UNIQUE (source_issue_id, target_issue_id)
);
```

### Step 3: Understand the `vector(1536)` type

`1536` is the dimension count for OpenAI's `text-embedding-3-small` and `text-embedding-ada-002` models.
Each embedding is an array of 1536 floating point numbers representing the semantic position of the text
in vector space. If you switch to a different model with different dimensions, you need a migration to
change the column type.

### Step 4: Verify migrations run on startup

Start the application (with Docker Compose running). In the logs you should see:

```
Flyway Community Edition ... by Redgate
Database: jdbc:postgresql://localhost:5432/deduplicator (PostgreSQL ...)
Successfully validated 1 migration (execution time 00:00.015s)
Creating Schema History table "public"."flyway_schema_history" ...
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "1 - create initial schema"
Successfully applied 1 migration to schema "public"
```

Then check in pgAdmin that the tables exist.

### Step 5: Inspect the schema history

```sql
SELECT * FROM flyway_schema_history;
```

You'll see a row for each applied migration with version, description, checksum, and execution time.

---

## Practical Tips

**Never edit an already-applied migration.** Flyway stores a checksum of each migration file.
If you modify a file that was already applied, Flyway will refuse to start with a `checksum mismatch` error.
Always create a new migration file for changes.

**Use `flyway:repair` when you're in trouble.** During development, if you mess up a migration
and need to reset: drop the database, recreate it, and Flyway will reapply everything from scratch.
In production, use `flyway repair` to fix the schema history table.

**Think about the UNIQUE constraint.** `uq_project_issue` prevents storing the same issue twice.
This matters in TASK-011 when we decide whether to INSERT or UPDATE an issue's embedding.

**Test your SQL in pgAdmin first.** Write and run the migration SQL in pgAdmin's query tool before
putting it in the file. Faster feedback loop.

**Learn `TIMESTAMPTZ` vs `TIMESTAMP`.** Always use `TIMESTAMPTZ` (timestamp with time zone) in PostgreSQL.
It stores values in UTC and converts to the session time zone on read. `TIMESTAMP` stores local time
with no zone info — a source of subtle bugs in distributed systems.
