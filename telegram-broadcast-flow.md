# Логика работы Telegram-рассылок

## Общая схема компонентов

```mermaid
flowchart TB
    subgraph Triggers["Триггеры запуска"]
        CRON[Крон: app:telegram:broadcast:send]
        CRON_REMINDER[Крон: app:telegram:broadcast:bonus-reminder]
        API_SEND["API: POST /broadcast/:id/send-now"]
        API_REMINDERS[API: POST /broadcast/send-bonus-reminders]
    end

    subgraph Service["TelegramBroadcastService"]
        SEND[sendBroadcast]
        REMINDERS[sendBonusReminders]
        ELIGIBLE[getEligibleCustomers]
        SEND_MSG[sendMessageToCustomer]
        TEMPLATE[processMessageTemplate]
    end

    subgraph Data["Данные"]
        SETTINGS[(TelegramBroadcastSettings)]
        LOG[(TelegramBroadcastLog)]
        CUSTOMERS[(Customers)]
    end

    CRON --> SEND
    API_SEND --> SEND
    CRON_REMINDER --> REMINDERS
    API_REMINDERS --> REMINDERS

    SEND --> ELIGIBLE
    SEND --> SEND_MSG
    SEND --> TEMPLATE
    REMINDERS --> SEND_MSG
    REMINDERS --> TEMPLATE

    ELIGIBLE --> CUSTOMERS
    ELIGIBLE --> LOG
    SEND --> LOG
    SEND --> SETTINGS
```

## Основная рассылка (sendBroadcast)

```mermaid
flowchart TD
    START([Запуск рассылки]) --> CHECK_ENABLED{Рассылка<br/>включена?}
    CHECK_ENABLED -->|Нет| ERR_DISABLED[Ошибка: рассылка отключена]
    CHECK_ENABLED -->|Да| CHECK_MSG{Текст<br/>сообщения задан?}
    CHECK_MSG -->|Нет| ERR_MSG[Ошибка: текст не задан]
    CHECK_MSG -->|Да| GET_CUSTOMERS[Получить eligible клиентов<br/>getEligibleCustomers]

    GET_CUSTOMERS --> EMPTY{Клиенты<br/>найдены?}
    EMPTY -->|Нет| RETURN_EMPTY[Возврат: sent=0]
    EMPTY -->|Да| LOOP_START[Для каждого клиента]

    LOOP_START --> LOAD_ENTITIES[Загрузить Customer + Order]
    LOAD_ENTITIES --> PROCESS_TEMPLATE[Подставить переменные в шаблон<br/>processMessageTemplate]
    PROCESS_TEMPLATE --> SEND_TG[Отправить в Telegram<br/>sendMessageToCustomer]

    SEND_TG --> RESULT{Результат<br/>отправки}
    RESULT -->|Успех| LOG_SENT[Лог: STATUS_SENT]
    RESULT -->|Заблокировал бота| LOG_BLOCKED[Лог: STATUS_BLOCKED]
    RESULT -->|Ошибка| LOG_FAILED[Лог: STATUS_FAILED]

    LOG_SENT --> BONUS_CHECK{Бонусы<br/>включены?}
    BONUS_CHECK -->|Да| ACCRUE_BONUS[Начислить промо-бонусы<br/>Bonus::addPromoBonus]
    BONUS_CHECK -->|Нет| SAVE_LOG
    ACCRUE_BONUS --> SAVE_LOG[Сохранить лог в БД]

    LOG_BLOCKED --> SAVE_LOG
    LOG_FAILED --> SAVE_LOG

    SAVE_LOG --> DELAY[Задержка sendDelayMs]
    DELAY --> LOOP_START
    LOOP_START --> UPDATE_STATS[Обновить lastRunAt, lastRunSentCount]

    style ERR_DISABLED fill:#f99
    style ERR_MSG fill:#f99
    style RETURN_EMPTY fill:#fc9
    style LOG_SENT fill:#9f9
    style LOG_BLOCKED fill:#f96
    style LOG_FAILED fill:#f99
```

## Критерии отбора клиентов (getEligibleCustomers)

```mermaid
flowchart TD
    START([getEligibleCustomers]) --> BASE[Базовый запрос]
    BASE --> C1[✓ Есть telegram_user_id]
    C1 --> C2[✓ Не в черном списке]
    C2 --> C3[✓ O = последний заказ клиента]
    C3 --> C4[✓ Дата заказа <= NOW - daysAfterOrder]
    C4 --> BLOCK[Исключить: getBlockedCustomerIds<br/>клиенты с отправкой за последние daysBlockRepeat дней]
    BLOCK --> FILTER_CITY{Фильтр<br/>городов?}
    FILTER_CITY -->|Да| CITY[c.town_id IN filterCityIds]
    FILTER_CITY -->|Нет| FILTER_TYPE
    CITY --> FILTER_TYPE{Фильтр<br/>типа клиента?}
    FILTER_TYPE -->|Да| TYPE[c.type IN filterCustomerTypes]
    FILTER_TYPE -->|Нет| SORT
    TYPE --> SORT[ORDER BY order_created_at DESC]
    SORT --> LIMIT[LIMIT maxMessagesPerRun]
    LIMIT --> RETURN[Вернуть список]
```

## Напоминания о сгорании бонусов (sendBonusReminders)

```mermaid
flowchart TD
    START([sendBonusReminders]) --> GET_SETTINGS[Найти все активные настройки<br/>findEnabled]
    GET_SETTINGS --> LOOP_SETTINGS[Для каждой настройки]
    LOOP_SETTINGS --> CHECK_BONUS{Бонусы +<br/>напоминания включены?}
    CHECK_BONUS -->|Нет| NEXT_SETTINGS[Следующая настройка]
    CHECK_BONUS -->|Да| CHECK_TEXT{Текст<br/>напоминания задан?}
    CHECK_TEXT -->|Нет| NEXT_SETTINGS
    CHECK_TEXT -->|Да| FIND_LOGS[findLogsForBonusReminder<br/>условия: bonusAccrued=true,<br/>bonusReminderSentAt=NULL,<br/>bonusExpiresAt через N дней]

    FIND_LOGS --> LOOP_LOGS[Для каждого лога]
    LOOP_LOGS --> PROCESS["Подставить переменные, в т.ч. expiration_date"]
    PROCESS --> SEND[Отправить в Telegram]
    SEND --> SUCCESS{Успех?}
    SUCCESS -->|Да| SET_REMINDER[bonusReminderSentAt = NOW]
    SUCCESS -->|Нет| INC_FAILED[result.failed++]
    SET_REMINDER --> DELAY[Задержка 100ms]
    INC_FAILED --> DELAY
    DELAY --> LOOP_LOGS
    LOOP_LOGS --> NEXT_SETTINGS
```

## Запуск по крону (TelegramBroadcastSendCommand)

```mermaid
flowchart TD
    START([app:telegram:broadcast:send]) --> PARAMS{Параметры}
    PARAMS --> SETTINGS_ID{--settings-id?}
    SETTINGS_ID -->|Есть| SINGLE[Одна рассылка по ID]
    SETTINGS_ID -->|Нет| ALL[Все включённые рассылки<br/>findEnabled]

    SINGLE --> LOOP
    ALL --> LOOP[Для каждой рассылки]
    LOOP --> CHECK_TIME{scheduleTime ==<br/>текущее время?}
    CHECK_TIME -->|Нет, без --force| SKIP[Пропустить]
    CHECK_TIME -->|Да или --force| CHECK_ENABLED{Включена?}
    CHECK_ENABLED -->|Нет| SKIP
    CHECK_ENABLED -->|Да| COUNT[countEligibleCustomers]
    COUNT --> DRY{--dry-run?}
    DRY -->|Да| PREVIEW[Показать превью без отправки]
    DRY -->|Нет| SEND[sendBroadcast]
    SEND --> RESULT[Вывод: sent, failed, blocked]
```

## API эндпоинты

```mermaid
flowchart LR
    subgraph Management["Управление"]
        GET_LIST[GET /broadcast - список]
        POST_CREATE[POST /broadcast - создать]
        GET_ONE["GET /broadcast/:id"]
        POST_UPDATE["POST /broadcast/:id - обновить"]
        DELETE["DELETE /broadcast/:id"]
        TOGGLE["POST /broadcast/:id/toggle"]
    end

    subgraph PreviewStats["Предпросмотр и статистика"]
        PREVIEW["GET /broadcast/:id/preview"]
        STATS["GET /broadcast/:id/stats"]
        LOGS["GET /broadcast/:id/logs"]
    end

    subgraph SendGroup["Отправка"]
        SEND_NOW["POST /broadcast/:id/send-now"]
        SEND_TEST["POST /broadcast/:id/send-test"]
        SEND_REMINDERS[POST /broadcast/send-bonus-reminders]
        SEND_TEST_REMINDER["POST /broadcast/:id/send-test-bonus-reminder"]
    end

    subgraph Media["Медиа"]
        UPLOAD_IMG["POST /broadcast/:id/upload-image"]
        DELETE_IMG["DELETE /broadcast/:id/image"]
    end
```

## Переменные шаблона сообщения

| Переменная | Описание |
|------------|----------|
| `{name}` | Имя клиента (или «Клиент») |
| `{full_name}` | Полное имя |
| `{first_name}` | Имя |
| `{phone}` | Телефон |
| `{order_number}` | Номер последнего заказа |
| `{order_date}` | Дата заказа (ДД.ММ.ГГГГ) |
| `{bonus_amount}` | Количество бонусов |
| `{bonus_expiration_days}` | Срок действия (дней) |
| `{bonus_expiration_date}` | Дата сгорания |
| `{expiration_date}` | В напоминаниях: дата сгорания |

## Схема данных

```mermaid
erDiagram
    TelegramBroadcastSettings ||--o{ TelegramBroadcastLog : "имеет"
    Customer ||--o{ TelegramBroadcastLog : "получает"
    Order ||--o{ TelegramBroadcastLog : "связан с"

    TelegramBroadcastSettings {
        int id PK
        string name
        bool is_enabled
        int days_after_order
        int days_block_repeat
        text message_text
        string schedule_time
        bool bonus_enabled
        int bonus_amount
        int bonus_expiration_days
        bool bonus_reminder_enabled
        int bonus_reminder_days_before
        text bonus_reminder_text
        json filter_city_ids
        json filter_customer_types
        int send_delay_ms
        int max_messages_per_run
    }

    TelegramBroadcastLog {
        int id PK
        int settings_id FK
        int customer_id FK
        int order_id FK
        string status "sent/blocked/failed"
        text sent_message_text
        datetime sent_at
        bool bonus_accrued
        int bonus_amount
        datetime bonus_expires_at
        datetime bonus_reminder_sent_at
    }
```
