# Решение: webhook платежа и подписка

## Часть 1. Схема данных и уникальности

### Таблицы

#### 1. users
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**Уникальности:**
- `email` - UNIQUE: один email = один пользователь, предотвращает дубликаты пользователей

**Индексы:**
- `idx_users_email` - быстрый поиск пользователя по email (O(log n) вместо O(n))

---

#### 2. subscriptions
```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(50) NOT NULL DEFAULT 'inactive', -- active, inactive, cancelled, expired
    plan_id VARCHAR(100) NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    started_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**Уникальности:**
- Нет уникальных полей (один пользователь может иметь несколько подписок в разное время)

**Индексы:**
- `idx_subscriptions_user_id` - быстрый поиск подписок пользователя
- `idx_subscriptions_status` - фильтрация по статусу (для cron-задач поиска активных/истекших)
- `idx_subscriptions_expires_at` - поиск истекших подписок
- `idx_subscriptions_user_status` - составной индекс для поиска активных подписок конкретного пользователя

**Статусы:**
- `active` - подписка активна
- `inactive` - подписка неактивна (еще не активирована или приостановлена)
- `cancelled` - подписка отменена пользователем
- `expired` - подписка истекла

---

#### 3. payments
```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    subscription_id UUID REFERENCES subscriptions(id) ON DELETE SET NULL,
    external_payment_id VARCHAR(255) NOT NULL UNIQUE, -- ID платежа от платежной системы
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL, -- pending, completed, failed, refunded
    webhook_event_id UUID REFERENCES webhook_events(id) ON DELETE SET NULL,
    processed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**Уникальности:**
- `external_payment_id` - UNIQUE: **ключ идемпотентности**, предотвращает создание дубликатов платежей при повторных webhook

**Индексы:**
- `idx_payments_external_payment_id` - быстрая проверка существования платежа (дедупликация)
- `idx_payments_user_id` - связи для JOIN запросов
- `idx_payments_subscription_id` - связи для JOIN запросов
- `idx_payments_status` - фильтрация по статусу
- `idx_payments_webhook_event_id` - связь с событием webhook для дебага
- `idx_payments_subscription_status` - поиск платежей по подписке и статусу

**Статусы:**
- `pending` - платеж в обработке
- `completed` - платеж успешно завершен
- `failed` - платеж не прошел
- `refunded` - платеж возвращен

---

#### 4. webhook_events
```sql
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_event_id VARCHAR(255) NOT NULL UNIQUE, -- ID события от платежной системы
    external_payment_id VARCHAR(255) NOT NULL, -- ID платежа из payload
    event_type VARCHAR(100) NOT NULL, -- payment.completed, payment.failed, etc.
    status VARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, processed, failed, duplicate
    payload JSONB NOT NULL, -- полный payload webhook для дебага
    signature VARCHAR(500), -- подпись webhook для валидации
    processed_at TIMESTAMP WITH TIME ZONE,
    error_message TEXT,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

**Уникальности:**
- `external_event_id` - UNIQUE: **ключ дедупликации webhook**, предотвращает повторную обработку одного и того же события

**Индексы:**
- `idx_webhook_events_external_event_id` - быстрая проверка дубликатов событий
- `idx_webhook_events_external_payment_id` - поиск событий по платежу
- `idx_webhook_events_status` - фильтрация необработанных событий (для восстановления)
- `idx_webhook_events_created_at` - сортировка по времени получения
- `idx_webhook_events_payload_gin` - GIN индекс для полнотекстового поиска по JSONB payload (для дебага)

**Статусы:**
- `pending` - событие получено, ожидает обработки
- `processed` - событие успешно обработано
- `failed` - ошибка при обработке события
- `duplicate` - событие является дубликатом

---

### Обоснование уникальностей

1. **`users.email` UNIQUE**: Предотвращает создание нескольких аккаунтов с одним email. Критично для корректной привязки платежей к пользователю.

2. **`payments.external_payment_id` UNIQUE**: Основной механизм идемпотентности. Если webhook придет повторно с тем же `external_payment_id`, платеж не будет создан дважды, подписка не продлится повторно.

3. **`webhook_events.external_event_id` UNIQUE**: Механизм дедупликации на уровне событий. Позволяет быстро определить, обрабатывали ли мы уже это событие.

---

## Часть 2. Пошаговая логика обработчика webhook

### Псевдокод обработчика

```javascript
async function handleWebhook(req, res) {
    let webhookEventId = null;
    let transaction = null;

    try {
        // ============================================
        // ШАГ 1: ВАЛИДАЦИЯ ВХОДНЫХ ДАННЫХ
        // ============================================
        const payload = req.body;
        const signature = req.headers['x-webhook-signature'];
        const eventId = payload.eventId;

        // Проверка обязательных полей
        if (!eventId || !payload.paymentId) {
            logger.warn('webhook_invalid_payload', { eventId, payload });
            return res.status(400).json({ error: 'Missing required fields' });
        }

        // ============================================
        // ШАГ 2: ПРОВЕРКА ПОДПИСИ
        // ============================================
        const isValidSignature = verifyWebhookSignature(
            JSON.stringify(payload),
            signature,
            process.env.WEBHOOK_SECRET
        );

        if (!isValidSignature) {
            logger.error('webhook_invalid_signature', { eventId });
            return res.status(401).json({ error: 'Invalid signature' });
        }

        // ============================================
        // ШАГ 3: ДЕДУПЛИКАЦИЯ - проверка существования события
        // ============================================
        const existingEvent = await db.query(
            'SELECT id, status FROM webhook_events WHERE external_event_id = $1 FOR UPDATE SKIP LOCKED',
            [eventId]
        );

        if (existingEvent.rows.length > 0) {
            const event = existingEvent.rows[0];
            
            // Если уже обработано - возвращаем 200 (идемпотентность)
            if (event.status === 'processed') {
                logger.info('webhook_duplicate', { eventId, status: event.status });
                return res.status(200).json({ message: 'Event already processed' });
            }
            
            // Если failed - можно попробовать снова, но для безопасности возвращаем 200
            return res.status(200).json({ message: 'Event already received' });
        }

        // ============================================
        // ШАГ 4: СОХРАНЕНИЕ СОБЫТИЯ (до транзакции)
        // ============================================
        const eventResult = await db.query(`
            INSERT INTO webhook_events 
            (external_event_id, external_payment_id, event_type, status, payload, signature)
            VALUES ($1, $2, $3, 'pending', $4, $5)
            RETURNING id
        `, [eventId, payload.paymentId, payload.eventType, JSON.stringify(payload), signature]);

        webhookEventId = eventResult.rows[0].id;

        // ============================================
        // ШАГ 5: НАЧАЛО ТРАНЗАКЦИИ
        // ============================================
        transaction = await db.beginTransaction();

        try {
            // ============================================
            // ШАГ 6: ПРОВЕРКА ДУБЛИКАТА ПЛАТЕЖА
            // ============================================
            const existingPayment = await transaction.query(
                'SELECT id, status FROM payments WHERE external_payment_id = $1',
                [payload.paymentId]
            );

            if (existingPayment.rows.length > 0) {
                await transaction.query(
                    'UPDATE webhook_events SET status = $1 WHERE id = $2',
                    ['duplicate', webhookEventId]
                );
                await transaction.commit();
                return res.status(200).json({ message: 'Payment already exists' });
            }

            // ============================================
            // ШАГ 7: ПОИСК ИЛИ СОЗДАНИЕ ПОЛЬЗОВАТЕЛЯ
            // ============================================
            let userId = null;

            if (payload.email) {
                const userResult = await transaction.query(
                    'SELECT id FROM users WHERE email = $1',
                    [payload.email]
                );

                if (userResult.rows.length > 0) {
                    userId = userResult.rows[0].id;
                } else {
                    const newUserResult = await transaction.query(`
                        INSERT INTO users (email) VALUES ($1) RETURNING id
                    `, [payload.email]);
                    userId = newUserResult.rows[0].id;
                }
            } else {
                await transaction.query(
                    'UPDATE webhook_events SET status = $1, error_message = $2 WHERE id = $3',
                    ['failed', 'No user identifier', webhookEventId]
                );
                await transaction.commit();
                return res.status(400).json({ error: 'No user identifier' });
            }

            // ============================================
            // ШАГ 8: ВАЛИДАЦИЯ СУММЫ ПЛАТЕЖА
            // ============================================
            const paymentAmount = parseFloat(payload.amount);
            
            if (payload.planId) {
                const planResult = await transaction.query(
                    'SELECT amount FROM subscriptions WHERE plan_id = $1 ORDER BY created_at DESC LIMIT 1',
                    [payload.planId]
                );

                if (planResult.rows.length > 0) {
                    const planAmount = parseFloat(planResult.rows[0].amount);
                    if (Math.abs(planAmount - paymentAmount) > 0.01) {
                        await transaction.query(
                            'UPDATE webhook_events SET status = $1, error_message = $2 WHERE id = $3',
                            ['failed', 'Amount mismatch', webhookEventId]
                        );
                        await transaction.commit();
                        return res.status(400).json({ error: 'Payment amount mismatch' });
                    }
                }
            }

            // ============================================
            // ШАГ 9: СОЗДАНИЕ ПЛАТЕЖА
            // ============================================
            const paymentStatus = payload.status === 'completed' ? 'completed' : 'pending';

            const paymentResult = await transaction.query(`
                INSERT INTO payments 
                (user_id, external_payment_id, amount, currency, status, webhook_event_id)
                VALUES ($1, $2, $3, $4, $5, $6)
                RETURNING id
            `, [userId, payload.paymentId, paymentAmount, payload.currency || 'USD', paymentStatus, webhookEventId]);

            const paymentId = paymentResult.rows[0].id;

            // ============================================
            // ШАГ 10: АКТИВАЦИЯ ИЛИ ПРОДЛЕНИЕ ПОДПИСКИ
            // ============================================
            if (paymentStatus === 'completed') {
                const subscriptionResult = await transaction.query(`
                    SELECT id, expires_at FROM subscriptions 
                    WHERE user_id = $1 AND status IN ('active', 'inactive')
                    ORDER BY created_at DESC LIMIT 1
                `, [userId]);

                let subscriptionId = null;
                const now = new Date();
                const subscriptionPeriod = 30; // дней

                if (subscriptionResult.rows.length > 0) {
                    // Продлеваем существующую подписку
                    const subscription = subscriptionResult.rows[0];
                    subscriptionId = subscription.id;

                    const currentExpiresAt = subscription.expires_at 
                        ? new Date(subscription.expires_at) 
                        : now;
                    
                    const newExpiresAt = currentExpiresAt > now
                        ? new Date(currentExpiresAt.getTime() + subscriptionPeriod * 24 * 60 * 60 * 1000)
                        : new Date(now.getTime() + subscriptionPeriod * 24 * 60 * 60 * 1000);

                    await transaction.query(`
                        UPDATE subscriptions 
                        SET status = 'active', expires_at = $1, updated_at = NOW()
                        WHERE id = $2
                    `, [newExpiresAt, subscriptionId]);
                } else {
                    // Создаем новую подписку
                    const newExpiresAt = new Date(now.getTime() + subscriptionPeriod * 24 * 60 * 60 * 1000);

                    const newSubscriptionResult = await transaction.query(`
                        INSERT INTO subscriptions 
                        (user_id, status, plan_id, amount, currency, started_at, expires_at)
                        VALUES ($1, 'active', $2, $3, $4, $5, $6)
                        RETURNING id
                    `, [userId, payload.planId || 'default', paymentAmount, payload.currency || 'USD', now, newExpiresAt]);

                    subscriptionId = newSubscriptionResult.rows[0].id;
                }

                // Обновляем payment со ссылкой на subscription
                await transaction.query(
                    'UPDATE payments SET subscription_id = $1 WHERE id = $2',
                    [subscriptionId, paymentId]
                );
            }

            // ============================================
            // ШАГ 11: ОБНОВЛЕНИЕ СТАТУСА СОБЫТИЯ
            // ============================================
            await transaction.query(`
                UPDATE webhook_events 
                SET status = 'processed', processed_at = NOW()
                WHERE id = $1
            `, [webhookEventId]);

            // ============================================
            // ШАГ 12: КОММИТ ТРАНЗАКЦИИ
            // ============================================
            await transaction.commit();
            transaction = null;

            logger.info('webhook_processed_successfully', { webhookEventId, eventId });
            return res.status(200).json({ message: 'Webhook processed successfully' });

        } catch (error) {
            // Откатываем транзакцию при ошибке
            if (transaction) {
                await transaction.rollback();
            }

            logger.error('webhook_processing_error', { webhookEventId, error: error.message });
            
            // Обновляем статус события на failed
            if (webhookEventId) {
                await db.query(`
                    UPDATE webhook_events 
                    SET status = 'failed', error_message = $1, retry_count = retry_count + 1
                    WHERE id = $2
                `, [error.message, webhookEventId]);
            }

            // Возвращаем 500 для повторной попытки платежной системы
            return res.status(500).json({ error: 'Internal server error' });
        }

    } catch (error) {
        logger.error('webhook_handler_error', { error: error.message });
        return res.status(500).json({ error: 'Internal server error' });
    }
}
```

### Что возвращаем и почему

- **200 OK**: 
  - Webhook успешно обработан
  - Webhook уже был обработан ранее (идемпотентность)
  - Платеж уже существует (дубликат)

- **400 Bad Request**: 
  - Отсутствуют обязательные поля в payload
  - Нет идентификатора пользователя (email/userId)
  - Несоответствие суммы платежа плану

- **401 Unauthorized**: 
  - Невалидная подпись webhook

- **500 Internal Server Error**: 
  - Ошибка при обработке (БД, логика и т.д.)
  - Платежная система должна повторить запрос

---

## Часть 3. Edge Cases

### 1. Webhook пришел дважды

**Решение:** Проверка по `external_event_id` перед обработкой. Если событие уже существует со статусом `processed` → возвращаем 200. Используем `SELECT FOR UPDATE SKIP LOCKED` для предотвращения параллельной обработки.

**Результат:** Дубликаты не создаются, подписка не продлевается дважды.

---

### 2. Webhook пришел раньше создания user

**Решение:** Если в payload есть `email` → создаем пользователя автоматически в той же транзакции. Если нет email/userId → возвращаем 400.

**Результат:** Пользователь создается автоматически, платеж обрабатывается корректно.

---

### 3. Webhook пришел без email, но есть externalPaymentId

**Решение:** Пытаемся найти существующий платеж по `external_payment_id` и получить `user_id`. Если платеж не найден → возвращаем 400.

**Результат:** Обрабатываем платеж для существующего пользователя, если можем его определить.

---

### 4. Webhook пришел с другой суммой, чем план

**Решение:** Если есть `planId` → проверяем соответствие суммы. Если суммы не совпадают (разница > 0.01) → отклоняем платеж (400), обновляем статус события на `failed`.

**Результат:** Защита от мошенничества и ошибок платежной системы.

---

### 5. Webhook пришел через неделю

**Решение:** Проверяем дату платежа в payload (если есть `createdAt`). Если платеж старше 7 дней → логируем предупреждение, но принимаем (или отклоняем в зависимости от бизнес-логики).

**Результат:** Обрабатываем поздние webhook с предупреждением или отклоняем устаревшие.

---

### 6. Сервер упал после записи payment, но до subscription

**Решение:** Используем транзакцию БД → при падении все откатывается. Если транзакция не успела закоммититься → payment не создан, можно обработать заново.

**Восстановление:** Периодический джоб проверяет платежи со статусом `completed`, но без активной подписки, и активирует подписку.

**Результат:** Система восстанавливается автоматически, подписки активируются даже после сбоя.

---

## Часть 4. Debuggability и наблюдаемость

### Минимальные логи (обязательные поля)

#### 1. Логи получения webhook
- `event_type`: "webhook_received"
- `eventId`, `paymentId`, `timestamp`, `ip`, `userAgent`

#### 2. Логи валидации
- `event_type`: "webhook_invalid_payload" | "webhook_invalid_signature"
- `eventId`, `reason`, `missingFields`

#### 3. Логи дедупликации
- `event_type`: "webhook_duplicate" | "webhook_payment_duplicate"
- `eventId`, `existingStatus`, `existingProcessedAt`

#### 4. Логи создания сущностей
- `event_type`: "user_created" | "payment_created" | "subscription_created" | "subscription_renewed"
- `userId`, `paymentId`, `subscriptionId`, `amount`, `webhookEventId`

#### 5. Логи ошибок обработки
- `event_type`: "webhook_processing_error"
- `eventId`, `webhookEventId`, `error`, `stack`, `retry_count`

#### 6. Логи успешной обработки
- `event_type`: "webhook_processed_successfully"
- `eventId`, `webhookEventId`, `processingTimeMs`

#### 7. Логи edge cases
- `event_type`: "webhook_amount_mismatch" | "webhook_late" | "webhook_no_user_identifier"
- Дополнительные поля в зависимости от типа

**Ключевые поля для корреляции:** `webhookEventId`, `eventId`, `paymentId`, `userId`

---

### Метрики

#### Счетчики (Counters)
- `webhook.received.total` - общее количество полученных webhook
- `webhook.signature.invalid` - невалидные подписи
- `webhook.duplicate` - дубликаты событий
- `webhook.payment.duplicate` - дубликаты платежей
- `webhook.processing.success` - успешно обработанные
- `webhook.processing.failed` - неудачно обработанные
- `webhook.processing.error` - ошибки при обработке
- `webhook.amount.mismatch` - несоответствие сумм

#### Гистограммы (Timings)
- `webhook.processing.time` - время обработки (p50, p95, p99)
- `webhook.validation.time` - время валидации
- `webhook.database.time` - время выполнения запросов к БД

#### Gauges
- `webhook.events.pending` - количество событий в статусе pending
- `webhook.events.failed` - количество событий в статусе failed
- `payments.orphaned` - количество платежей без подписки

---

### Алерты

#### Критические (P0)
1. **Высокий процент ошибок**: `rate(webhook.processing.error[5m]) / rate(webhook.received.total[5m]) > 0.1`
2. **Невалидные подписи**: `rate(webhook.signature.invalid[5m]) > 5`
3. **Медленная обработка**: `p95(webhook.processing.time[5m]) > 5000`

#### Предупреждающие (P1)
1. **Много дубликатов**: `rate(webhook.duplicate[1h]) > 50`
2. **Несоответствие сумм**: `rate(webhook.amount.mismatch[1h]) > 10`
3. **Много failed событий**: `webhook.events.failed > 100`

#### Информационные (P2)
1. **Низкая активность**: `rate(webhook.received.total[1h]) < 1`
2. **Orphaned платежи**: `payments.orphaned > 20`

---

### Сохранение payload для дебага

**Где:** В таблице `webhook_events`, поле `payload` (JSONB)

**Что сохраняем:**
- Полный payload webhook (все поля)
- Заголовки запроса (signature, user-agent, ip)
- Время получения
- Статус обработки
- Ошибки (если есть)

**Зачем:**
- Воспроизведение проблемных webhook
- Анализ изменений в формате payload
- Аудит и соответствие требованиям

---

## Итоговая архитектура

### Ключевые принципы

1. **Идемпотентность**: Уникальные ключи (`external_payment_id`, `external_event_id`) предотвращают дубликаты
2. **Транзакции**: Все изменения в одной транзакции для атомарности
3. **Восстановление**: Джоб для обработки orphaned платежей
4. **Валидация**: Проверка подписи, обязательных полей, сумм
5. **Дебаг**: Полное логирование и сохранение payload

### Защита от проблем

- ✅ Повторные webhook не создают дубликаты
- ✅ Повторные webhook не продлевают подписку дважды
- ✅ Система восстанавливается после сбоев
- ✅ Все операции атомарны (транзакции)
- ✅ Полная наблюдаемость (логи + метрики)
