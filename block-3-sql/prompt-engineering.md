# Prompt Engineering для генерации SQL-запросов

генерация SQL-запросов через LLM с последующей проверкой в DBeaver на тестовой базе логистической тематики
(заказы, перевозчики, аукционы, ставки, категории ТС).

## Схема тестовой базы

| Таблица              | Назначение                       |
|----------------------|----------------------------------|
| `cities`             | справочник городов               | 
| `customers`          | заказчики (грузовладельцы)       |
| `carriers`           | перевозчики                      |
| `drivers`            | водители                         |
| `vehicle_categories` | справочник категорий ТС          |
| `orders`             | заказы на перевозку              |
| `bids`               | ставки перевозчиков по аукционам |
| `invoices`           | счета по заказам                 |


---

## Промпт 1 (базовый JOIN + фильтр)

```text
В PostgreSQL есть таблица orders (customer_id, carrier_id, price, status, loading_date)
и customers (id, name). Напиши запрос, который покажет все заказы со статусом
'delivered' за последние 60 дней, с именем клиента, отсортировано по цене
по убыванию.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT c.name, o.price, o.loading_date
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'delivered'
  AND o.loading_date >= CURRENT_DATE - INTERVAL '60 days'
ORDER BY o.price DESC;
```

---

## Промпт 2 (агрегация + исключение по флагу)

```text
Таблицы carriers (id, name, rating, is_blacklisted) и orders (carrier_id, price).
Посчитай суммарную выручку каждого перевозчика и его рейтинг, отсортируй
по выручке по убыванию, исключи перевозчиков из чёрного списка.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT c.name, c.rating, SUM(o.price) AS total_revenue
FROM carriers c
JOIN orders o ON o.carrier_id = c.id
WHERE c.is_blacklisted = false
GROUP BY c.id, c.name, c.rating
ORDER BY total_revenue DESC;
```

---

## Промпт 3 (оконные функции, разрыв между ставками)

```text
Таблица bids (order_id, carrier_id, bid_amount, is_winner). Для каждого заказа
покажи разницу между ставкой победителя и второй по величине (по возрастанию)
ставкой — то есть на сколько выгоднее оказался выигрыш по сравнению со следующим
конкурентом.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
WITH ranked_bids AS (
    SELECT
        order_id,
        bid_amount,
        is_winner,
        ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY bid_amount ASC) AS rnk
    FROM bids
)
SELECT
    w.order_id,
    w.bid_amount AS winning_bid,
    r2.bid_amount AS second_lowest_bid,
    r2.bid_amount - w.bid_amount AS gap
FROM ranked_bids w
JOIN ranked_bids r2 ON r2.order_id = w.order_id AND r2.rnk = 2
WHERE w.rnk = 1;
```

---

## Промпт 4 (несоответствие бизнес-логики — победила не минимальная ставка)

```text
Таблица bids (order_id, carrier_id, bid_amount, is_winner) — найди заказы-аукционы,
где победителем (is_winner = true) была выбрана НЕ ставка с минимальной суммой
среди всех ставок по этому заказу.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT b.order_id, b.bid_amount AS winning_bid, min_b.min_bid
FROM bids b
JOIN (
    SELECT order_id, MIN(bid_amount) AS min_bid
    FROM bids
    GROUP BY order_id
) min_b ON min_b.order_id = b.order_id
WHERE b.is_winner = true
  AND b.bid_amount > min_b.min_bid;
```

---

## Промпт 5 (демпинг относительно собственного среднего)

```text
Таблица bids (order_id, carrier_id, bid_amount). Найди перевозчиков, у которых
хотя бы одна ставка оказалась ниже их собственной средней ставки по всем
остальным заказам минимум на 20%.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT b.carrier_id, b.order_id, b.bid_amount, avg_b.avg_bid
FROM bids b
JOIN (
    SELECT carrier_id, AVG(bid_amount) AS avg_bid
    FROM bids
    GROUP BY carrier_id
) avg_b ON avg_b.carrier_id = b.carrier_id
WHERE b.bid_amount < avg_b.avg_bid * 0.8;
```

---

## Промпт 6 (группировка по месяцу и типу заказа)

```text
Сгруппируй заказы по месяцу подачи (created_at) и типу заказа (order_type),
посчитай количество заказов и среднюю ставку с НДС (base_rate_with_vat)
по каждой группе, используя DATE_TRUNC.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT
    DATE_TRUNC('month', created_at) AS month,
    order_type,
    COUNT(*) AS orders_count,
    ROUND(AVG(base_rate_with_vat), 2) AS avg_rate_with_vat
FROM orders
GROUP BY DATE_TRUNC('month', created_at), order_type
ORDER BY month, order_type;
```

---

## Промпт 7 (перевозчики из чёрного списка, выигравшие аукцион)

```text
Покажи перевозчиков с is_blacklisted = true, которые всё равно выиграли хотя бы
один аукцион (bids.is_winner = true) за последние 90 дней, с указанием заказа
и суммы ставки.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT c.name, b.order_id, b.bid_amount, b.submitted_at
FROM bids b
JOIN carriers c ON c.id = b.carrier_id
WHERE c.is_blacklisted = true
  AND b.is_winner = true
  AND b.submitted_at >= CURRENT_DATE - INTERVAL '90 days';
```

---

## Промпт 8 (неоплаченная задолженность по клиентам)

```text
Таблица invoices (order_id, amount, due_date, paid_at) и orders (customer_id).
Найди клиентов, у которых сумма неоплаченных счетов (paid_at IS NULL) превышает
200 000 рублей суммарно, с указанием количества таких счетов.
```

**Ответ LLM:**
_(вставить после прогона)_

**Результат в DBeaver / заметки:**
_(вставить после прогона)_

**Эталон (для самопроверки):**
```sql
SELECT cu.name, COUNT(*) AS unpaid_invoices, SUM(i.amount) AS total_unpaid
FROM invoices i
JOIN orders o ON o.id = i.order_id
JOIN customers cu ON cu.id = o.customer_id
WHERE i.paid_at IS NULL
GROUP BY cu.id, cu.name
HAVING SUM(i.amount) > 200000
ORDER BY total_unpaid DESC;
```