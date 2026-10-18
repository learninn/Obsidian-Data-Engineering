---
tags:
  - Completed
---
**Dimensional Modeling (размерностное моделирование)** — техника проектирования аналитических структур данных, при которой информация делится на два типа:

- **[[Facts]] (факты)** — измеримые события бизнеса (продажа, клик, платёж)
    
- **[[Dimensions]] (измерения)** — контекст, описывающий факты (кто, что, где, когда, как)

**Главная цель:** сделать данные **понятными бизнесу и быстрыми для BI**. Не «правильными» с точки зрения нормализации, а удобными для анализа.

**Ключевой принцип:** модель должна отражать **как бизнес задаёт вопросы**, а не **как данные хранятся в источниках**.

### Почему не 3NF

| Критерий            | 3NF (нормализованная)     | Dimensional (звезда)          |
| ------------------- | ------------------------- | ----------------------------- |
| Цель                | Убрать дубли, целостность | Понятность, скорость запросов |
| JOIN'ов в запросе   | Много (5–15)              | Мало (1–3)                    |
| Понятность бизнесу  | Низкая                    | Высокая                       |
| Скорость BI-запроса | Медленно                  | Быстро                        |
| Избыточность        | Минимум                   | Есть (денормализация)         |
| Изменения           | Гибко                     | Требует продумывания          |

### Пример

Бизнес спрашивает: "Выручка по городам за март".

**3NF:**

```sql
SELECT city.name, SUM(oi.price * oi.qty)
FROM order_item oi
JOIN orders o ON oi.order_id = o.id
JOIN client c ON o.client_id = c.id
JOIN client_address ca ON c.id = ca.client_id
JOIN address a ON ca.address_id = a.id
JOIN city ON a.city_id = city.id
JOIN date d ON o.date_id = d.id
WHERE d.month = 3 AND d.year = 2026
GROUP BY city.name;
```

6 JOIN'ов. Аналитик должен знать структуру.

**Dimensional:**

```sql
SELECT c.city, SUM(f.revenue)
FROM fact_sales f
JOIN dim_client c ON f.client_sk = c.client_sk
JOIN dim_date d ON f.date_sk = d.date_sk
WHERE d.month = 3 AND d.year = 2026
GROUP BY c.city;
```

2 JOIN'а. Понятно даже новичку.


