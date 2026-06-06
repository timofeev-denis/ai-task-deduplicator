# AI GitLab Issue Deduplication Service

## 1. Цель продукта

Сервис автоматически обнаруживает дубликаты и связанные баги в GitLab Issues.

После создания или изменения Issue в GitLab сервис:

1. Получает webhook.
2. Анализирует содержимое Issue.
3. Находит похожие ранее зарегистрированные Issues.
4. Определяет тип связи:

   * DUPLICATE
   * RELATED
   * DIFFERENT
5. Создает связи между задачами через GitLab API.
6. Добавляет метки для дубликатов.
7. Уведомляет автора Issue по email.

---

# 2. Бизнес-ценность

Сервис позволяет:

* уменьшить количество дубликатов багов;
* сократить время анализа новых дефектов;
* улучшить качество статистики в GitLab;
* ускорить работу тестировщиков и разработчиков;
* повторно использовать накопленные знания команды.

---

# 3. Целевая аудитория

## Основная

* QA Engineers
* Test Engineers
* QA Leads

## Вторичная

* Developers
* Team Leads
* Engineering Managers
* Product Owners

---

# 4. Основные сценарии использования

## Сценарий 1. Создание нового бага

Тестировщик создает Issue:

```text
После обновления приложения
вход через Google перестал работать.
```

Сервис:

* получает webhook;
* ищет похожие задачи;
* обнаруживает дубликат;
* создает связь;
* добавляет label;
* отправляет уведомление.

---

## Сценарий 2. Изменение существующего бага

Пользователь дополняет описание.

Сервис повторно выполняет анализ.

---

## Сценарий 3. Обнаружение связанных проблем

Система определяет, что задачи не являются дубликатами, но относятся к одной области.

Создается связь типа:

```text
RELATED
```

---

## Сценарий 4. Уведомление автора

Автор получает email:

```text
Для вашей задачи обнаружены похожие Issues.

Созданные связи:
- Issue #123
- Issue #147

Статус:
DUPLICATE
```

---

# 5. Архитектура

## Архитектурный стиль

Модульный монолит.

Использовать Spring Modulith.

---

## Модули

```text
webhook
gitlab
issue-processing
similarity
notification
observability
shared
```

---

## Поток обработки

```text
GitLab
    ↓
Webhook
    ↓
IssueReceivedEvent
    ↓
Issue Loader
    ↓
Embedding Generation
    ↓
Vector Search
    ↓
Top Candidates
    ↓
LLM Classification
    ↓
Relation Decision
    ↓
GitLab Update
    ↓
Email Notification
```

---

# 6. Технологический стек

## Backend

* Java 25
* Spring Boot 4
* Spring Modulith

## Database

* PostgreSQL
* pgvector (vector similarity extension)
* Flyway

## AI

* Spring AI
* OpenAI Embeddings
* OpenAI Chat Model

## Integrations

* GitLab REST API
* SMTP
* Mailpit (local email testing)

## Observability

* Spring Actuator
* Micrometer
* OpenTelemetry
* Prometheus

## Documentation

* SpringDoc OpenAPI

## Build

* Maven
* Docker Compose

---

# 7. Бизнес-правила

## Similarity Search

Находятся TOP-10 наиболее похожих задач.

---

## Classification

LLM должна вернуть одно из значений:

```text
DUPLICATE
RELATED
DIFFERENT
```

---

## Duplicate

Если классификация:

```text
DUPLICATE
```

то:

* создать GitLab Link;
* добавить label:

```text
duplicate-detected
```

* отправить email.

---

## Related

Если классификация:

```text
RELATED
```

то:

* создать GitLab Link;
* отправить email.

---

## Different

Никаких действий.

---

# EPIC-1 Project Bootstrap

---

## TASK-001 Create Spring Boot Project

### Goal

Создать базовый проект.

### Requirements

Подключить:

* Spring Web
* Spring Data JPA
* PostgreSQL
* Flyway
* Spring Modulith
* Spring AI
* Spring Actuator

### Definition of Done

Приложение запускается.

---

## TASK-002 Configure Docker Compose

### Goal

Поднять инфраструктуру локально.

### Requirements

Создать контейнеры:

* postgres (с расширением pgvector)
* pgadmin
* mailpit

### Definition of Done

Инфраструктура запускается через:

```bash
docker compose up
```

Mailpit UI доступен на `http://localhost:8025`.
SMTP доступен на порту `1025`.

---

# EPIC-2 Persistence

---

## TASK-003 Create Database Schema

### Goal

Создать базовую схему БД.

### Tables

#### processed_issue

Поля:

```text
id
gitlab_project_id
gitlab_issue_iid
title
description
embedding  (vector(1536) — pgvector, OpenAI ada-002 / text-embedding-3-small dimensions)
created_at
updated_at
```

Индекс:

```sql
CREATE INDEX ON processed_issue USING hnsw (embedding vector_cosine_ops);
```

---

#### issue_relation

Поля:

```text
id
source_issue_id
target_issue_id
relation_type
confidence
created_at
```

---

### Definition of Done

Миграции создают таблицы.

---

## TASK-004 Create JPA Entities

### Goal

Создать сущности.

### Definition of Done

Entity и Repository созданы.

---

# EPIC-3 GitLab Integration

---

## TASK-005 Create GitLab HTTP Client

### Goal

Реализовать интеграцию через HTTP Service Client.

### Requirements

Создать:

```java
GitLabClient
```

через:

```java
@HttpExchange
```

### Operations

* get issue
* create relation
* add label

### Definition of Done

Клиент работает.

---

## TASK-006 Create GitLab Configuration

### Goal

Настроить:

```text
base-url
private-token
```

через application.yml.

---

## TASK-007 Load Issue Details

### Goal

Получать полное описание Issue из GitLab.

### Definition of Done

Issue успешно загружается.

---

# EPIC-4 Webhooks

---

## TASK-008 Create Webhook Endpoint

### Endpoint

```http
POST /api/webhooks/gitlab/issues
```

### Goal

Получать события:

```text
issue opened
issue updated
```

### Security

Валидировать заголовок GitLab:

```text
X-Gitlab-Token: <secret>
```

Секрет задаётся в `application.yml`. При несовпадении — вернуть `403 Forbidden`.

### Definition of Done

* Webhook принимает запросы.
* Запросы без валидного токена отклоняются.

---

## TASK-009 Publish Domain Event

### Event

```java
IssueReceivedEvent
```

### Goal

Отвязать webhook от бизнес-логики.

### Async Strategy

Использовать **Spring Modulith Event Publication Registry**:

1. Webhook handler публикует `IssueReceivedEvent` и возвращает `200 OK` немедленно.
2. Событие сохраняется в БД (в той же транзакции) через реестр публикаций.
3. Listener (`@ApplicationModuleListener`) обрабатывает событие асинхронно.
4. После успешной обработки — событие помечается как завершённое.
5. При сбое или рестарте — незавершённые события повторяются автоматически.

Это гарантирует:
* Быстрый ответ GitLab (webhook timeout не превышается).
* Надёжность: событие не теряется при сбое.

### Definition of Done

* Событие публикуется и сохраняется в реестре.
* Listener обрабатывает событие асинхронно.
* При рестарте незавершённые события повторяются.

---

# EPIC-5 Embeddings

---

## TASK-010 Integrate Embedding Model

### Goal

Генерировать embeddings для Issues.

### Input

```text
title
description
```

### Output

Vector embedding.

### Definition of Done

Embedding успешно генерируется.

---

## TASK-011 Persist Embeddings

### Goal

Сохранять embeddings.

### Definition of Done

Embedding хранится в БД.

---

# EPIC-6 Similarity Search

---

## TASK-012 Implement Similarity Search

### Goal

Находить похожие Issues.

### Requirements

Возвращать:

```text
TOP-10
```

наиболее похожих задач.

### Definition of Done

Поиск работает.

---

## TASK-013 Create Candidate Ranking

### Goal

Отсортировать результаты по similarity score.

### Output

```text
Issue
Score
```

---

# EPIC-7 AI Classification

---

## TASK-014 Integrate Chat Model

### Goal

Подключить LLM.

### Definition of Done

Можно отправлять промпты.

---

## TASK-015 Classify Issue Pair

### Goal

Определить отношение между двумя задачами.

### Output

```json
{
  "classification": "DUPLICATE",
  "confidence": 0.91
}
```

### Definition of Done

Классификация работает.

---

## TASK-016 Process Candidate List

### Goal

Проверить TOP-10 кандидатов.

### Definition of Done

Получен итоговый список:

```text
DUPLICATE
RELATED
```

---

# EPIC-8 GitLab Actions

---

## TASK-017 Create Issue Links

### Goal

Создавать связи между задачами.

### Input

```text
Source Issue
Target Issue
Relation Type
```

### Definition of Done

Связи создаются через GitLab API.

---

## TASK-018 Add Duplicate Label

### Goal

Добавлять метку:

```text
duplicate-detected
```

### Definition of Done

Метка появляется в GitLab.

---

## TASK-019 Store Decision History

### Goal

Сохранять результаты классификации.

### Definition of Done

История доступна в БД.

---

# EPIC-9 Email Notifications

---

## TASK-020 Configure SMTP

### Goal

Настроить отправку email.

### Definition of Done

Письма отправляются.

---

## TASK-021 Send Duplicate Notification

### Goal

Уведомлять автора Issue.

### Email Content

* Issue номер
* найденные дубликаты
* confidence
* добавленные связи

### Definition of Done

Письмо доставляется.

---

## TASK-022 Send Related Notification

### Goal

Уведомлять о связанных задачах.

### Definition of Done

Письмо отправляется.

---

# EPIC-10 Observability

---

## TASK-023 Enable Actuator

### Endpoints

```text
/actuator/health
/actuator/info
/actuator/metrics
```

---

## TASK-024 Add Custom Metrics

### Metrics

```text
issues_processed_total
duplicates_detected_total
related_detected_total
emails_sent_total
```

---

## TASK-025 Add OpenTelemetry Tracing

### Goal

Трассировать полный pipeline:

```text
Webhook
→ Embedding
→ Search
→ Classification
→ GitLab Update
→ Email
```

---

# EPIC-11 Testing

---

## TASK-026 Unit Tests

Покрыть:

* Similarity Service
* Classification Service
* GitLab Service

---

## TASK-027 Integration Tests

Покрыть:

* Webhook Processing
* GitLab Integration
* Email Notifications

---

# EPIC-12 Documentation

---

## TASK-028 Configure OpenAPI

### Goal

Документировать API.

---

## TASK-029 Create README

### Должен содержать

* архитектуру;
* запуск;
* настройку GitLab;
* настройку OpenAI;
* настройку SMTP;
* описание pipeline.

---

# MVP Completion Criteria

Система считается готовой, если:

1. GitLab отправляет webhook.
2. Issue загружается через GitLab API.
3. Генерируется embedding.
4. Находятся похожие задачи.
5. LLM классифицирует кандидатов.
6. Для DUPLICATE создаются связи и label.
7. Для RELATED создаются связи.
8. Автор получает email.
9. Все шаги наблюдаемы через метрики и tracing.
10. Проект запускается локально через Docker Compose.
