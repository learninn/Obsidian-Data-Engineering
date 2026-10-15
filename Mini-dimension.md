**Mini-dimension** — отдельное измерение, которое хранит **только быстро меняющиеся (волатильные) атрибуты** (часто называемые **demographics** или **behavioral attributes**), связанное с фактом напрямую. Основное и mini-измерение согласовываются через ETL и таблицу фактов. 

### Идея: демография — это комбинация значений, а не сам клиент

- **Медленные атрибуты** → остаются в `dim_client` **(SCD2)**
    
- **Быстрые атрибуты** → выносятся в `dim_client_demographics` **(SCD1/SCD2)**
    
- **Факт** ссылается на **оба** измерения

### Схема

**Без mini-dimension:**

```text
 staging ──── dim_client (всё вместе, SCD2) ──── fact_sales 
```

**С mini-dimension:**

```text

 staging ────┬── dim_client (медленные, SCD2)────────────────────┬──fact_sales
             └── dim_client_demographics (быстрые, SCD1/SCD2)────┘
```


### Бизнес-контекст

Магазин хочет анализировать:

- Продажи по возрастным группам
    
- Продажи по доходу
    
- Продажи по кредитному рейтингу
    
- Продажи по лояльности
    
- Продажи по жизненному циклу


При этом клиенты меняют эти атрибуты часто.

**dim_client (только медленные, SCD2):

```text
client_sk | client_id | name | age_band | income_band | valid_from | valid_to
1         | C001      | Иван | 26-35    | 30-60k      | 2024-01-01 | 9999-12-31
```

**dim_client_demographics (быстрые, SCD1/SCD2):**

```text
demo_sk | credit | tier   | stage   | valid_from | valid_to
5       | good   | silver | active  | 2024-01-01 | 2024-04-01
6       | good   | gold   | active  | 2024-04-01 | 2024-07-01
7       | good   | gold   | churned | 2024-07-01 | 2024-10-01
8       | good   | gold   | active  | 2024-10-01 | 9999-12-31
```

**fact_sales:**

```text
date_sk | client_sk | demo_sk | product_sk | revenue
...
2024-02-15 | 1 | 5 | 100 | 5000   -- покупка, когда tier=silver
2024-05-20 | 1 | 6 | 101 | 3000   -- покупка, когда tier=gold
2024-08-10 | 1 | 7 | 102 | 2000   -- покупка, когда churned
2024-11-05 | 1 | 8 | 103 | 7000   -- покупка, когда снова active
```

**Что даёт:**

- `dim_client` — 1 строка вместо 4
    
- `dim_client_demographics` — компактная, уникальные комбинации
    
- Факт точно знает состояние на момент покупки
    
- Анализ по демографии работает


### Запросы

**Продажи по tier:**
```sql
SELECT d.loyalty_tier, SUM(f.revenue)
FROM fact_sales f
JOIN dim_client_demographics d ON f.demo_sk = d.demo_sk
GROUP BY d.loyalty_tier;
```

**Продажи по возрасту и tier:**
```sql
SELECT c.age_band, d.loyalty_tier, SUM(f.revenue)
FROM fact_sales f
JOIN dim_client c ON f.client_sk = c.client_sk
JOIN dim_client_demographics d ON f.demo_sk = d.demo_sk
GROUP BY c.age_band, d.loyalty_tier;
```

**Продажи по клиенту с его демографией на момент покупки:**
```sql
SELECT c.client_name, d.loyalty_tier, d.lifecycle_stage, SUM(f.revenue)
FROM fact_sales f
JOIN dim_client c ON f.client_sk = c.client_sk
JOIN dim_client_demographics d ON f.demo_sk = d.demo_sk
GROUP BY c.client_name, d.loyalty_tier, d.lifecycle_stage;
```


### Типичные кандидаты на mini-dimention:

| Домен     | Быстрые атрибуты                                                   |
| --------- | ------------------------------------------------------------------ |
| Клиент    | age_band, income_band, credit_score, loyalty_tier, lifecycle_stage |
| Товар     | price_band, popularity_rank, stock_status                          |
| Сотрудник | role, department, performance_band                                 |
| Магазин   | traffic_band, size_band, format                                    |

## Варианты реализации

### Классический (Kimball)

- **dim_client** — медленные атрибуты, SCD2
    
- **dim_client_demographics** — быстрые атрибуты, SCD1 или SCD2
    
- **fact_sales** — client_sk + demo_sk

### С banding (диапазоны)

Вместо точных значений — диапазоны:

```text
age_band: 18-25, 26-35, 36-45, 46-55, 55+
income_band: <30k, 30-60k, 60-100k, 100k+
credit_band: poor, fair, good, excellent
```

**Плюс:** резко сокращает число уникальных комбинаций.  
**Минус:** теряется точность, обязательно требуется **справочник band'ов** —  **golden record** домена. **Вычисление band из точного значения** — это **логика трансформации**, которая должна **ссылаться** на справочник.

### SCD2 в mini-dimension

Если нужно отслеживать историю изменений демографии:

```text
demo_sk | credit | tier | valid_from | valid_to | is_current
5       | good   | silver | 2024-01-01 | 2024-04-01 | false
6       | good   | gold   | 2024-04-01 | 2024-07-01 | false
```

**Плюс:** полная история.  
**Минус:** растёт быстрее, чем SCD1.

### Гибрид с основной dim_client

Иногда demo_sk хранится **в dim_client** (SCD2), а факт ссылается только на client_sk.

```text
dim_client: client_sk, client_id, name, demo_sk, valid_from, valid_to
fact_sales: client_sk (demo_sk подтягивается через JOIN)
```

**Плюс:** проще загрузка факта.  
**Минус:** dim_client растёт при изменении демографии.

## Приколы и грабли

### Проблемы проектирования

| Проблема                                  | Пример                         | Решение                     |
| ----------------------------------------- | ------------------------------ | --------------------------- |
| Слишком много комбинаций в mini-dimention | 1 млн demo_sk                  | Banding (диапазоны)         |
| Слишком мало комбинаций                   | 5 demo_sk                      | Возможно, не нужна mini-dim |
| Атрибуты не используются                  | demo_sk есть, но не в запросах | Убрать                      |
| Смешение медленных и быстрых              | age в demo, name в client      | Чёткое разделение           |
| Нет banding                               | Точные значения                | Banding                     |

### Проблемы загрузки

| Проблема                | Пример                               | Решение              |
| ----------------------- | ------------------------------------ | -------------------- |
| demo_sk не найден       | Комбинации нет                       | Создать новую строку |
| Late-arriving demo      | Факт пришёл, demo нет                | [[Inferred member]]  |
| Изменение задним числом | Demo изменилась, история пересобрана | SCD2 в demo          |
