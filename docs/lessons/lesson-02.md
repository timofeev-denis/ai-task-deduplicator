# Lesson 02: Configure Docker Compose

## What We're Building

A `docker-compose.yml` that starts PostgreSQL (with pgvector), pgAdmin, and Mailpit with a single command.
Spring Boot 4 will auto-detect and connect to these containers during development.

---

## Technologies

### Docker Compose
Defines and runs multi-container applications. You describe services in a YAML file and Docker manages
their lifecycle. Key concepts:
- **Service**: a container definition (image, ports, environment variables, volumes).
- **Volume**: persistent storage that survives container restarts.
- **Network**: containers in the same Compose file can reach each other by service name.

### PostgreSQL with pgvector
`pgvector` is a PostgreSQL extension that adds a `vector` data type and similarity search operators
(`<=>` for cosine distance, `<->` for L2 distance, `<#>` for inner product). It also provides
HNSW and IVFFlat index types for fast approximate nearest-neighbor search.

We use the `pgvector/pgvector:pg17` Docker image which has the extension pre-installed.

### pgAdmin
A web-based PostgreSQL management UI. Useful for inspecting tables, running queries,
and visualizing the `event_publication` and `processed_issue` tables during development.

### Mailpit
A local SMTP server + web UI for catching outgoing emails during development.
Instead of sending real emails, your app sends to Mailpit (port 1025),
and you view them in the browser at `http://localhost:8025`.
No real email addresses needed, no spam risk.

### Spring Boot Docker Compose Integration
Spring Boot 4 has a built-in `spring-boot-docker-compose` module. When it detects a
`docker-compose.yml` in the project root, it automatically starts the containers before
the application and stops them when the application stops. This means you never need
to run `docker compose up` manually during development.

---

## Step-by-Step

### Step 1: Add the Docker Compose support dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-docker-compose</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

This is only active in development — `optional` prevents it from being inherited by dependents.

### Step 2: Create `docker-compose.yml` in the project root

```yaml
services:

  postgres:
    image: pgvector/pgvector:pg17
    container_name: deduplicator-postgres
    environment:
      POSTGRES_DB: deduplicator
      POSTGRES_USER: deduplicator
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U deduplicator"]
      interval: 5s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: deduplicator-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres

  mailpit:
    image: axllent/mailpit:latest
    container_name: deduplicator-mailpit
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI
    environment:
      MP_SMTP_AUTH_ACCEPT_ANY: 1
      MP_SMTP_AUTH_ALLOW_INSECURE: 1

volumes:
  postgres_data:
```

### Step 3: Configure Spring Boot to use Mailpit SMTP

In `application.yml`:

```yaml
spring:
  mail:
    host: localhost
    port: 1025
    username: test
    password: test
    properties:
      mail:
        smtp:
          auth: false
          starttls:
            enable: false
```

### Step 4: Verify everything starts

```bash
docker compose up
```

Check:
- PostgreSQL: `psql -h localhost -U deduplicator -d deduplicator`
- pgAdmin: open `http://localhost:5050` — login with `admin@admin.com` / `admin`
- Mailpit: open `http://localhost:8025`

### Step 5: Connect pgAdmin to PostgreSQL

In pgAdmin UI:
1. Right-click "Servers" → Register → Server
2. Name: `deduplicator`
3. Connection tab: Host = `postgres` (service name), Port = `5432`, User = `deduplicator`, Password = `secret`

Note: use the service name `postgres`, not `localhost`, because pgAdmin runs inside Docker.

---

## Practical Tips

**Use named volumes, not bind mounts, for databases.** Named volumes (`postgres_data:`) are managed
by Docker and survive container recreation. Bind mounts (e.g., `./data:/var/lib/postgresql/data`)
work too but can cause permission issues on Linux.

**Always add a `healthcheck` to your database container.** Without it, Spring Boot may try to connect
before PostgreSQL is ready and fail. The `healthcheck` + `depends_on` pattern ensures ordering.

**Learn `docker compose` commands you'll use daily:**
```bash
docker compose up -d          # start in background
docker compose down           # stop and remove containers
docker compose down -v        # also delete volumes (wipes DB data)
docker compose logs postgres  # tail logs for a specific service
docker compose ps             # show running services
```

**The pgvector image matters.** The standard `postgres:17` image does NOT have pgvector.
You must use `pgvector/pgvector:pg17` or install the extension manually.
Verify it's available after startup:
```sql
SELECT * FROM pg_available_extensions WHERE name = 'vector';
```
