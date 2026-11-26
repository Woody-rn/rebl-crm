# ADR-001: Core Architecture

**Status:** Accepted (Revised)
**Date:** 2026-11-26
**Deciders:** Lead Backend Developer
**Tech Stack:** Java 21, Spring Boot 4.2.0, PostgreSQL 16+ (recommended 18), React/TS, MinIO

## 1. Context and Problem Statement

ReblCRM — AI-first, Telegram-native CRM с универсальным ядром и вертикальными шаблонами.
Цель — минимизировать ручной ввод через голосовой/текстовый AI-парсинг и обеспечить гибкость через кастомные поля.

Требования:
- Мультиарендность с изоляцией данных на уровне БД
- Производительность 10 000+ пользователей
- Расширяемость (новые вертикали без миграций схемы)

## 2. Decisions

### 2.1. Modular Monolith + Pragmatic Architecture

Модульный монолит в базовом пакете `ru.npepub.reblcrm`:
- `core` — безопасность, `TenantContext`, базовые сущности, общие исключения
- `contacts` — контакты, компании, кастомные поля
- `deals` — сделки, воронки
- `tasks` — задачи, напоминания
- `scheduling` — расписание, бронирования, внешние календари
- `messaging` — Telegram-бот, мессенджеры, Outbox-уведомления
- `ai` — AI-интеграции (LLM, NLP, распознавание голоса)

**Подход внутри модулей:**
- **Hexagonal (Ports & Adapters)** — только для модулей с внешними интеграциями (`ai`, `scheduling`, `messaging`).
- **Layered (Controller → Service → Repository)** — для стандартных CRUD-модулей (`contacts`, `deals`, `tasks`), чтобы избежать лишнего шаблонного маппинга на этапе MVP.

### 2.2. Multi-Tenancy: PostgreSQL RLS

Shared Database, Shared Schema, Row-Level Security.

- Все тенанты в схеме `public`
- Каждая бизнес-таблица имеет `tenant_id UUID NOT NULL`
- Безопасная политика RLS (не падает при пустом значении в фоновых задачах):
  ```sql
  USING (tenant_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid)
  ```
- Передача контекста: `tenant_id` из JWT → `SELECT set_config('app.current_tenant', ?, true)` (локально на время транзакции)
- Единая роль БД `app_user` с `FORCE ROW LEVEL SECURITY`

### 2.3. Hybrid Data Model: Columns + JSONB

- Базовые поля — типизированные колонки с B-Tree индексами
- Кастомные поля — `custom_data JSONB` + `custom_field_definitions`
- GIN-индексы на `custom_data`
- Валидация JSONB-структуры в доменном/сервисном слое перед сохранением

### 2.4. Soft Deletes

- Все бизнес-сущности имеют `deleted_at TIMESTAMPTZ NULL`.
- На уровне Hibernate используется `@SQLRestriction("deleted_at IS NULL")`.
- Для уникальных полей (например, `email`) используются **частичные уникальные индексы** (`WHERE deleted_at IS NULL`), чтобы удалённые записи не блокировали создание новых.

### 2.5. Virtual Threads (Java 21) & Context Propagation

- Включены виртуальные потоки (`spring.threads.virtual.enabled=true`).
- Для `@Async` используется `Executors.newVirtualThreadPerTaskExecutor()`.
- Вместо ручного `ThreadLocal` текущий `tenantId` и `userId` извлекаются из `SecurityContextHolder` (который корректно пробрасывается в виртуальных потоках Spring Boot 4) либо передаются явно в фоновые задачи.

### 2.6. Technology Stack

| Компонент | Выбор |
| :--- | :--- |
| Backend | Java 21 + Spring Boot 4.2.0 |
| Database | PostgreSQL 16+ (рекомендуется 18) |
| ORM | Hibernate 7.x |
| Migrations | Liquibase (YAML) |
| Cache | Caffeine |
| Storage | MinIO (S3) |
| API Docs | springdoc-openapi 3.x |
| Testing | JUnit 5 + Testcontainers + ArchUnit |
| Build | Gradle 8.x |

### 2.7. Indexing Strategy for Multi-Tenant Tables

Все индексы на бизнес-таблицах с `tenant_id` должны быть **составными**, начиная с `tenant_id`:
- `(tenant_id, phone)` вместо `(phone)`
- `(tenant_id, email)` вместо `(email)`
- `(tenant_id, telegram_chat_id)` вместо `(telegram_chat_id)`

**Обоснование:** PostgreSQL RLS неявно добавляет `WHERE tenant_id = ?` ко всем запросам.
Составной индекс покрывает и RLS-фильтр, и бизнес-запрос одним B-Tree-сканированием.

**Исключения:**
- GIN-индексы на JSONB (специфичная структура)
- Индексы на `deleted_at` (используются редко)
- Уникальные частичные индексы (например, `uq_users_email_active`)

## 3. Consequences

### Positive
- ✅ Минимум бойлерплейта в простых CRUD-модулях
- ✅ Жёсткая изоляция данных тенантов на уровне PostgreSQL
- ✅ Отсутствие утечек `ThreadLocal` на виртуальных потоках

### Risks
- ⚠️ В `@Scheduled` задачах `app.current_tenant` пуст — фоновые джобы должны явно выставлять тенанта для каждой итерации.
