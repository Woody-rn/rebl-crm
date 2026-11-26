# ADR-003: Monolith vs Microservices Strategy

**Status:** Accepted (Revised)
**Date:** 2026-11-26

## 1. Context

ReblCRM стартует как модульный монолит. Фиксируем правила изоляции модулей и критерии перехода на микросервисы.

## 2. Decision: Modular Monolith First

- Команда 1–3 разработчика, нагрузка до 10 000 пользователей.
- Единая транзакционность в PostgreSQL, деплой одним JAR-файлом.

## 3. Rules for Modular Monolith

### 3.1. Общение между модулями

Модули не вызывают репозитории друг друга напрямую. Взаимодействие идёт через:
1. Публичные доменные события (`ApplicationEventPublisher` + `@TransactionalEventListener`).
2. Публичные сервисные фасады/контракты модуля (пакет `..api..`), возвращающие DTO, а не JPA-сущности.

### 3.2. Изоляция сущностей

```java
// ПЛОХО: прямая связь Entity между модулями
public class Deal {
    @ManyToOne
    private Contact contact;
}

// ХОРОШО: ссылка по идентификатору
public class Deal {
    private UUID contactId;
}
```

### 3.3. ArchUnit тесты

Модули могут зависеть только от общего модуля `core` и публичных контрактов (`..api..`, `..events..`) других модулей:

```java
@AnalyzeClasses(packages = "ru.npepub.reblcrm", importOptions = ImportOption.DoNotIncludeTests.class)
public class ArchitectureTest {

    @ArchTest
    static final ArchRule modules_should_only_access_public_apis_and_core =
        classes().that().resideInAnyPackage("ru.npepub.reblcrm.contacts..", "ru.npepub.reblcrm.deals..", "ru.npepub.reblcrm.scheduling..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage(
                "java..", "jakarta..", "org.springframework..", "org.slf4j..", "lombok..",
                "ru.npepub.reblcrm.core..",
                "ru.npepub.reblcrm..api..",
                "ru.npepub.reblcrm..events..",
                // Разрешаем доступ внутри собственного модуля
                "ru.npepub.reblcrm.contacts..",
                "ru.npepub.reblcrm.deals..",
                "ru.npepub.reblcrm.scheduling.."
            );
}
```

## 4. Migration Criteria

Переход к выделению отдельных микросервисов рассматривается при достижении любого из порогов:
- Команда: **10+ разработчиков**
- Нагрузка: **100 000+ активных пользователей** или **10 000+ RPS**
- Необходимость независимого масштабирования тяжёлого модуля (например, `ai` или `messaging`)

## 5. Migration Plan

1. Выделить модуль в отдельный Spring Boot сервис.
2. Заменить вызовы `..api..` интерфейсов на декларативный HTTP-клиент Spring (`@HttpExchange` + `RestClient`).
3. Заменить `ApplicationEventPublisher` на брокер сообщений (Kafka / RabbitMQ) поверх существующей таблицы `outbox_events`.
