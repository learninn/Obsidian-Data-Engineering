---
tags:
  - Completed
---
**Role-Playing Dimension** (ролевое измерение) — это **одно и то же измерение**, которое используется в **одной таблице фактов несколько раз**, но в **разных ролях**.

**Ключевая идея:** физически это **одна таблица**, но логически она играет разные роли в зависимости от того, на какой FK факта она навешена.

Самый частый пример — `dim_date`.

## Классический пример: dim_date

В `fact_sales` есть несколько дат:
```sql
CREATE TABLE fact_sales (
    sale_id          BIGINT,
    order_date_sk    INT,   -- дата заказа
    ship_date_sk     INT,   -- дата отгрузки
    delivery_date_sk INT,   -- дата доставки
    client_sk        INT,
    product_sk       INT,
    revenue          DECIMAL(18,2)
);
```

**Все три FK ссылаются на одну и ту же `dim_date`:**

```sql
                    ┌─────────────────┐
                    │    dim_date     │
                    │  date_sk (PK)   │
                    │  date           │
                    │  month, quarter │
                    │  year, ...      │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        order_date_sk   ship_date_sk   delivery_date_sk
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   fact_sales    │
                    └─────────────────┘
```

**Одна таблица — три роли:**

- `dim_date` как **дата заказа**
    
- `dim_date` как **дата отгрузки**
    
- `dim_date` как **дата доставки**

## Как это выглядит в SQL

```sql
SELECT 
    od.date  AS order_date,
    sd.date  AS ship_date,
    dd.date  AS delivery_date,
    f.revenue
FROM fact_sales f
JOIN dim_date od ON f.order_date_sk    = od.date_sk   -- роль: дата заказа
JOIN dim_date sd ON f.ship_date_sk     = sd.date_sk   -- роль: дата отгрузки
JOIN dim_date dd ON f.delivery_date_sk = dd.date_sk   -- роль: дата доставки
WHERE od.year = 2026 AND od.month = 3;
```

**Три JOIN'а на одну таблицу**, но с разными алиасами (`od`, `sd`, `dd`).

### Аналитика

**«Сколько заказов отгружено с опозданием?»**

```sql
SELECT 
    od.date AS order_date,
    sd.date AS ship_date,
    DATEDIFF(day, od.date, sd.date) AS delay_days
FROM fact_sales f
JOIN dim_date od ON f.order_date_sk = od.date_sk
JOIN dim_date sd ON f.ship_date_sk  = sd.date_sk
WHERE DATEDIFF(day, od.date, sd.date) > 3;
```

**Без role-playing** пришлось бы делать три разные таблицы `dim_order_date`, `dim_ship_date`, `dim_delivery_date` — дублирование.


## Другие примеры
### dim_store в разных ролях

В `fact_sales`:

- `origin_store_sk` — магазин отправки
    
- `destination_store_sk` — магазин получения
    

Все → `dim_store`.

### dim_client в разных ролях

В `fact_sales`:

- `buyer_sk` — клиент-покупатель
    
- `seller_sk` — клиент-продавец (для маркетплейса)
    
- `recommender_sk` — клиент, который рекомендовал

Все три → `dim_client`.

### dim_employee в разных ролях

В `fact_sales`:

- `salesperson_sk` — продавец
    
- `cashier_sk` — кассир
    
- `manager_sk` — менеджер

Все три → `dim_employee`.


## Почему это Role-Playing, а не просто «одна таблица»

**Разница:**

|Обычное измерение|Role-Playing|
|---|---|
|Один FK в факте|Несколько FK в факте|
|Одна роль|Несколько ролей|
|`dim_client` → `client_sk`|`dim_date` → `order_date_sk`, `ship_date_sk`, `delivery_date_sk`|

**Ключевое:** одна физическая таблица, **разные логические роли** в одном факте.

## Реализация: view

Когда BI не умеет делать несколько JOIN'ов на одну таблицу с разными алиасами, role-dimention реализовывается как несколько view для JOIN в таблице фактов:

```sql
CREATE VIEW dim_order_date AS 
    SELECT date_sk, date, month, quarter, year 
    FROM dim_date;

CREATE VIEW dim_ship_date AS 
    SELECT date_sk, date, month, quarter, year 
    FROM dim_date;
```
