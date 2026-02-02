# Edge Cases - Обработка граничных ситуаций

## 1. Webhook пришел дважды

**Ситуация:** Платежная система отправила один и тот же webhook дважды (с одинаковым `eventId`).

**Решение:**
- Проверка по `external_event_id` в таблице `webhook_events` перед обработкой
- Если событие уже существует со статусом `processed` → возвращаем 200 (идемпотентность)
- Если событие существует со статусом `failed` → можно попробовать обработать снова, но для безопасности возвращаем 200
- Если событие существует со статусом `pending` → возможна параллельная обработка, используем SELECT FOR UPDATE для блокировки

**Код:**
```sql
SELECT id, status FROM webhook_events 
WHERE external_event_id = $1 
FOR UPDATE SKIP LOCKED;
```

**Результат:** Дубликаты не создаются, подписка не продлевается дважды.

---

## 2. Webhook пришел раньше создания user

**Ситуация:** Платеж пришел для пользователя, которого еще нет в системе.

**Решение:**
- Если в payload есть `email` → создаем пользователя автоматически в той же транзакции
- Если в payload есть `userId` → используем его (если платежная система передает)
- Если нет ни email, ни userId → возвращаем 400 с ошибкой "No user identifier"

**Код:**
```javascript
if (payload.email) {
    // Создаем пользователя если не существует
    const userResult = await transaction.query(
        'INSERT INTO users (email) VALUES ($1) ON CONFLICT (email) DO UPDATE SET email = users.email RETURNING id',
        [payload.email]
    );
}
```

**Результат:** Пользователь создается автоматически, платеж обрабатывается корректно.

---

## 3. Webhook пришел без email, но есть externalPaymentId

**Ситуация:** В payload нет email, но есть `externalPaymentId` (ID платежа).

**Решение:**
- Пытаемся найти существующий платеж по `external_payment_id`
- Если платеж найден → получаем `user_id` из платежа
- Если платеж не найден → возвращаем 400 (не можем определить пользователя)

**Альтернатива:** Если платежная система передает `userId` или `customerId` → используем его.

**Код:**
```javascript
if (!payload.email && !payload.userId) {
    // Пытаемся найти по существующему платежу
    const existingPayment = await transaction.query(
        'SELECT user_id FROM payments WHERE external_payment_id = $1',
        [payload.externalPaymentId]
    );
    
    if (existingPayment.rows.length > 0) {
        userId = existingPayment.rows[0].user_id;
    } else {
        return res.status(400).json({ error: 'No user identifier' });
    }
}
```

**Результат:** Обрабатываем платеж для существующего пользователя, если можем его определить.

---

## 4. Webhook пришел с другой суммой, чем план

**Ситуация:** Сумма в webhook не совпадает с суммой плана подписки.

**Решение:**
- Если есть `planId` или `subscriptionId` → проверяем соответствие суммы
- Если суммы не совпадают (разница > 0.01) → отклоняем платеж (400)
- Логируем предупреждение с ожидаемой и полученной суммой
- Обновляем статус webhook_event на `failed` с описанием ошибки

**Код:**
```javascript
if (Math.abs(plan.amount - paymentAmount) > 0.01) {
    await transaction.query(
        'UPDATE webhook_events SET status = $1, error_message = $2 WHERE id = $3',
        ['failed', `Amount mismatch: expected ${plan.amount}, got ${paymentAmount}`, webhookEventId]
    );
    return res.status(400).json({ error: 'Payment amount mismatch' });
}
```

**Результат:** Защита от мошенничества и ошибок платежной системы.

---

## 5. Webhook пришел через неделю

**Ситуация:** Webhook пришел с большой задержкой (например, через неделю после платежа).

**Решение:**
- Проверяем дату платежа в payload (если есть поле `createdAt` или `timestamp`)
- Если платеж старше N дней (например, 7) → можно либо:
  - Отклонить как устаревший (400)
  - Принять, но пометить как "late" в логах
- Если даты нет в payload → принимаем (платежная система решила отправить позже)

**Код:**
```javascript
if (payload.createdAt) {
    const paymentDate = new Date(payload.createdAt);
    const daysDiff = (Date.now() - paymentDate.getTime()) / (1000 * 60 * 60 * 24);
    
    if (daysDiff > 7) {
        logger.warn('webhook_late', {
            webhookEventId,
            daysDiff,
            paymentDate: paymentDate.toISOString()
        });
        // Можно отклонить или принять с предупреждением
    }
}
```

**Результат:** Обрабатываем поздние webhook с предупреждением или отклоняем устаревшие.

---

## 6. Сервер упал после записи payment, но до subscription

**Ситуация:** В процессе обработки сервер упал после создания payment, но до обновления subscription.

**Решение:**
- Используем транзакцию БД → при падении все откатывается
- Если транзакция не успела закоммититься → payment не создан, можно обработать заново
- Если транзакция закоммитилась частично → БД сама откатит изменения

**Восстановление:**
- Периодический джоб проверяет платежи со статусом `completed`, но без активной подписки
- Джоб находит такие платежи и активирует/продлевает подписку

**Код восстановления:**
```javascript
// Периодический джоб (запускается каждые 5 минут)
async function recoverFailedSubscriptions() {
    const orphanPayments = await db.query(`
        SELECT p.id, p.user_id, p.amount, p.currency, p.created_at
        FROM payments p
        WHERE p.status = 'completed'
        AND p.subscription_id IS NULL
        AND p.created_at > NOW() - INTERVAL '1 hour'
    `);
    
    for (const payment of orphanPayments.rows) {
        // Активируем подписку для платежа
        await activateSubscriptionForPayment(payment);
    }
}
```

**Результат:** Система восстанавливается автоматически, подписки активируются даже после сбоя.

---

## Дополнительные Edge Cases

### 7. Параллельная обработка одного webhook

**Решение:** Использовать `SELECT FOR UPDATE SKIP LOCKED` для блокировки строки.

### 8. Webhook пришел для отмененной подписки

**Решение:** Проверяем статус подписки перед продлением. Если `cancelled` → создаем новую подписку.

### 9. Несколько платежей для одной подписки одновременно

**Решение:** Используем транзакции и блокировки. Первый платеж продлевает подписку, остальные помечаются как дубликаты.

### 10. Webhook пришел для несуществующего плана

**Решение:** Если `planId` не найден → создаем подписку с переданным `planId` и суммой из платежа (или отклоняем, в зависимости от бизнес-логики).
