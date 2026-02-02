# Observability - Логи и Метрики

## Минимальные логи (обязательные поля)

### 1. Логи получения webhook

**Уровень:** INFO

**Поля:**
- `event_type`: "webhook_received"
- `eventId`: ID события от платежной системы
- `paymentId`: ID платежа из payload
- `timestamp`: время получения
- `ip`: IP адрес отправителя
- `userAgent`: User-Agent заголовок
- `payload_size`: размер payload в байтах

**Пример:**
```json
{
  "level": "info",
  "event_type": "webhook_received",
  "eventId": "evt_1234567890",
  "paymentId": "pay_9876543210",
  "timestamp": "2024-01-15T10:30:00Z",
  "ip": "192.168.1.1",
  "userAgent": "Stripe/1.0",
  "payload_size": 1024
}
```

---

### 2. Логи валидации

**Уровень:** WARN (невалидные данные), ERROR (невалидная подпись)

**Поля:**
- `event_type`: "webhook_invalid_payload" | "webhook_invalid_signature"
- `eventId`: ID события
- `paymentId`: ID платежа
- `reason`: причина отклонения
- `missingFields`: список отсутствующих полей (для payload)
- `signature_preview`: первые 20 символов подписи (для безопасности)

**Пример:**
```json
{
  "level": "warn",
  "event_type": "webhook_invalid_payload",
  "eventId": "evt_1234567890",
  "paymentId": null,
  "reason": "Missing required fields",
  "missingFields": ["paymentId", "amount"]
}
```

---

### 3. Логи дедупликации

**Уровень:** INFO

**Поля:**
- `event_type`: "webhook_duplicate" | "webhook_payment_duplicate"
- `eventId`: ID события
- `paymentId`: ID платежа
- `existingStatus`: статус существующего события/платежа
- `existingProcessedAt`: время предыдущей обработки

**Пример:**
```json
{
  "level": "info",
  "event_type": "webhook_duplicate",
  "eventId": "evt_1234567890",
  "paymentId": "pay_9876543210",
  "existingStatus": "processed",
  "existingProcessedAt": "2024-01-15T10:25:00Z"
}
```

---

### 4. Логи создания сущностей

**Уровень:** INFO

**Поля:**
- `event_type`: "user_created" | "payment_created" | "subscription_created" | "subscription_renewed"
- `userId`: ID пользователя
- `paymentId`: ID платежа (для payment/subscription)
- `subscriptionId`: ID подписки (для subscription)
- `amount`: сумма платежа
- `currency`: валюта
- `webhookEventId`: ID события webhook
- `expiresAt`: дата окончания подписки (для subscription)

**Пример:**
```json
{
  "level": "info",
  "event_type": "subscription_renewed",
  "userId": "user_abc123",
  "paymentId": "pay_9876543210",
  "subscriptionId": "sub_xyz789",
  "amount": 29.99,
  "currency": "USD",
  "webhookEventId": "evt_1234567890",
  "expiresAt": "2024-02-15T10:30:00Z"
}
```

---

### 5. Логи ошибок обработки

**Уровень:** ERROR

**Поля:**
- `event_type`: "webhook_processing_error" | "webhook_handler_error"
- `eventId`: ID события
- `paymentId`: ID платежа
- `webhookEventId`: ID события в БД
- `error`: сообщение об ошибке
- `stack`: stack trace (только в development/staging)
- `retry_count`: количество попыток обработки

**Пример:**
```json
{
  "level": "error",
  "event_type": "webhook_processing_error",
  "eventId": "evt_1234567890",
  "paymentId": "pay_9876543210",
  "webhookEventId": "evt_db_abc123",
  "error": "Database connection timeout",
  "stack": "...",
  "retry_count": 2
}
```

---

### 6. Логи успешной обработки

**Уровень:** INFO

**Поля:**
- `event_type`: "webhook_processed_successfully"
- `eventId`: ID события
- `paymentId`: ID платежа
- `webhookEventId`: ID события в БД
- `processingTimeMs`: время обработки в миллисекундах

**Пример:**
```json
{
  "level": "info",
  "event_type": "webhook_processed_successfully",
  "eventId": "evt_1234567890",
  "paymentId": "pay_9876543210",
  "webhookEventId": "evt_db_abc123",
  "processingTimeMs": 245
}
```

---

### 7. Логи edge cases

**Уровень:** WARN

**Поля:**
- `event_type`: "webhook_amount_mismatch" | "webhook_late" | "webhook_no_user_identifier"
- `eventId`: ID события
- `paymentId`: ID платежа
- Дополнительные поля в зависимости от типа edge case

**Пример:**
```json
{
  "level": "warn",
  "event_type": "webhook_amount_mismatch",
  "eventId": "evt_1234567890",
  "paymentId": "pay_9876543210",
  "expectedAmount": 29.99,
  "receivedAmount": 19.99,
  "expectedCurrency": "USD",
  "receivedCurrency": "USD"
}
```

---

## Метрики (обязательные)

### 1. Счетчики (Counters)

**Метрики:**
- `webhook.received.total` - общее количество полученных webhook
- `webhook.signature.invalid` - количество невалидных подписей
- `webhook.duplicate` - количество дубликатов событий
- `webhook.payment.duplicate` - количество дубликатов платежей
- `webhook.processing.success` - успешно обработанные webhook
- `webhook.processing.failed` - неудачно обработанные webhook
- `webhook.processing.error` - ошибки при обработке
- `webhook.amount.mismatch` - несоответствие сумм
- `webhook.handler.error` - ошибки на уровне handler

**Теги:**
- `event_type`: тип события (payment.completed, payment.failed, etc.)
- `payment_provider`: платежная система (stripe, paypal, etc.)

---

### 2. Гистограммы (Histograms/Timings)

**Метрики:**
- `webhook.processing.time` - время обработки webhook (в миллисекундах)
- `webhook.validation.time` - время валидации (в миллисекундах)
- `webhook.database.time` - время выполнения запросов к БД (в миллисекундах)

**Процентили:** p50, p95, p99

---

### 3. Gauges

**Метрики:**
- `webhook.events.pending` - количество событий в статусе pending
- `webhook.events.failed` - количество событий в статусе failed
- `payments.orphaned` - количество платежей без подписки (для восстановления)

---

## Алерты

### 1. Критические алерты (P0 - немедленное реагирование)

**Алерт:** Высокий процент ошибок обработки webhook
- **Условие:** `rate(webhook.processing.error[5m]) / rate(webhook.received.total[5m]) > 0.1` (более 10% ошибок)
- **Действие:** Уведомление в Slack/PagerDuty, проверка логов

**Алерт:** Невалидные подписи webhook
- **Условие:** `rate(webhook.signature.invalid[5m]) > 5` (более 5 невалидных подписей за 5 минут)
- **Действие:** Проверка секрета webhook, возможная компрометация

**Алерт:** Медленная обработка webhook
- **Условие:** `p95(webhook.processing.time[5m]) > 5000` (95-й процентиль > 5 секунд)
- **Действие:** Проверка производительности БД, возможные проблемы с индексами

---

### 2. Предупреждающие алерты (P1 - реагирование в течение часа)

**Алерт:** Много дубликатов webhook
- **Условие:** `rate(webhook.duplicate[1h]) > 50` (более 50 дубликатов в час)
- **Действие:** Проверка настроек платежной системы, возможные проблемы с retry логикой

**Алерт:** Несоответствие сумм платежей
- **Условие:** `rate(webhook.amount.mismatch[1h]) > 10` (более 10 несоответствий в час)
- **Действие:** Проверка конфигурации планов, возможные проблемы с ценообразованием

**Алерт:** Много событий в статусе failed
- **Условие:** `webhook.events.failed > 100` (более 100 failed событий)
- **Действие:** Запуск джоба восстановления, проверка логов

---

### 3. Информационные алерты (P2 - мониторинг)

**Алерт:** Низкая активность webhook
- **Условие:** `rate(webhook.received.total[1h]) < 1` (менее 1 webhook в час в рабочее время)
- **Действие:** Проверка интеграции с платежной системой

**Алерт:** Много orphaned платежей
- **Условие:** `payments.orphaned > 20` (более 20 платежей без подписки)
- **Действие:** Запуск джоба восстановления подписок

---

## Структура логов для дебага

### Сохранение payload

**Где:** В таблице `webhook_events`, поле `payload` (JSONB)

**Что сохраняем:**
- Полный payload webhook (все поля)
- Заголовки запроса (signature, user-agent, ip)
- Время получения
- Статус обработки
- Ошибки (если есть)

**Зачем:**
- Воспроизведение проблемных webhook
- Анализ изменений в формате payload от платежной системы
- Аудит и соответствие требованиям

---

### Корреляция логов

**Ключевые поля для корреляции:**
- `webhookEventId` - связывает все логи одного webhook
- `eventId` - внешний ID события от платежной системы
- `paymentId` - ID платежа
- `userId` - ID пользователя
- `subscriptionId` - ID подписки

**Пример запроса в логах:**
```
webhookEventId="evt_db_abc123" OR eventId="evt_1234567890"
```

---

## Дашборды

### 1. Общий дашборд webhook

**Графики:**
- Rate получения webhook (за последний час/день)
- Rate успешной обработки vs ошибок
- Процент дубликатов
- Время обработки (p50, p95, p99)
- Количество событий по статусам (pending, processed, failed)

### 2. Дашборд ошибок

**Графики:**
- Типы ошибок (stacked bar chart)
- Тренд ошибок во времени
- Топ ошибок по частоте
- Связь ошибок с временем обработки

### 3. Дашборд восстановления

**Графики:**
- Количество orphaned платежей
- Количество восстановленных подписок
- Время восстановления
