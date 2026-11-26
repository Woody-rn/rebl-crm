# ADR-002: Scheduling Module and External Integrations

**Status:** Accepted (Revised)
**Date:** 2026-11-26

## 1. Context

Модуль расписания (бронирования, уроки, визиты) + двусторонняя синхронизация с внешними календарями.

## 2. Decisions

### 2.1. External Calendar Port (MVP Scope)

```java
public interface ExternalCalendarPort {
    List<BusyInterval> getBusyIntervals(UUID userId, Instant from, Instant to);
    void subscribeToChanges(UUID userId, String webhookUrl);
}
```

**Адаптеры на этапе MVP (покрывают ~90% аудитории):**
- `GoogleCalendarAdapter` — OAuth 2.0 + Push Webhooks
- `YandexCalendarAdapter` — CalDAV + Polling

*(Outlook и Mail.ru подключаются на следующем этапе через реализацию того же порта без изменения ядра модуля).*

### 2.2. Timezone Handling

- Все timestamps в БД хранятся строго в UTC (`TIMESTAMPTZ`).
- `tenants.timezone`, `users.timezone`, `contacts.timezone`, `event_types.timezone` — строки в формате IANA (например, `Europe/Moscow`, `Asia/Krasnoyarsk`).
- Недельные правила доступности хранятся как `local time + weekday + tz`, конвертация в UTC происходит только при расчёте свободных слотов.

### 2.3. Double Booking Prevention

Защита от гонок (race condition) на уровне PostgreSQL через Exclusion Constraint:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings ADD CONSTRAINT bookings_no_overlap
  EXCLUDE USING gist (
    tenant_id   WITH =,
    resource_id WITH =,
    tstzrange(starts_at, ends_at, '[)') WITH &&
  ) WHERE (status IN ('PENDING', 'CONFIRMED'));
```

### 2.4. Public Booking Zone

- Эндпоинты `/api/v1/public/{tenantSlug}/{eventSlug}/slots` работают без JWT.
- Используется единая роль БД `app_user` (без создания второй роли и второго пула соединений).
- Фильтр/сервис резолвит `tenant_id` по `tenantSlug` из таблицы `tenants` и выставляет `set_config('app.current_tenant', tenantId, true)` перед чтением слотов или созданием брони.
- Защита от спама: Rate Limiting (Bucket4j / Caffeine) по `IP + tenantSlug`.

### 2.5. Message Broker: NOT NEEDED (MVP)

Не используем RabbitMQ/Kafka на старте.

**Альтернативы в рамках монолита:**
- Polling календарей → `@Scheduled` + Virtual Threads
- Гарантированная отправка уведомлений (Telegram/Email) → паттерн **Transactional Outbox** (таблица `outbox_events` + фоновый воркер)
- Межмодульная связь → `ApplicationEventPublisher` (`@TransactionalEventListener`)

### 2.6. Booking Status Model

```text
PENDING → CONFIRMED → COMPLETED | CANCELED | NO_SHOW | RESCHEDULED
```
