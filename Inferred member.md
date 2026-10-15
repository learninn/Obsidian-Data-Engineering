**Inferred member** — это **временная строка-заглушка** в таблице измерений, которая создаётся, когда **факт ссылается на элемент измерения, который ещё не загружен в DWH**.

**Ключевая идея:** мы знаем **бизнес-ключ** (например, `client_id = C001`), но ещё не знаем **атрибуты** (имя, город, демография). Создаём «скелет» с известным ключом и заполняем атрибуты позже.

**Терминология Кимбалла:** «late arriving dimensions» или «early arriving facts».

## Реализация
### Создание inferred member при загрузке факта

```text
Источник (биллинг): 
    sale_id = 1001, client_id = C999, sale_date = 2026-03-15, amount = 5000

ETL:
    1. Ищет client_id = C999 в dim_client
    2. Не находит
    3. Создаёт inferred member:
       client_sk = 1254 (новый, уникальный)
       client_id = C999
       client_name = 'Unknown' (или NULL)
       city = 'Unknown' (или NULL)
       is_inferred = true
    4. Записывает факт: (date_sk, client_sk=1254, amount=5000)
```

**Важно:** `client_sk` уникален для `C999`. Если придёт ещё один факт с C999, он использует тот же `client_sk = 1254`.

### Обновление inferred member

Когда реальные данные о клиенте C999 наконец приходят:

```text
Источник (CRM): 
    client_id = C999, client_name = 'Иван', city = 'Москва'

ETL:
    1. Находит inferred member с client_id = C999
    2. UPDATE dim_client SET 
         client_name = 'Иван',
         city = 'Москва',
         is_inferred = false
       WHERE client_id = 'C999' AND is_inferred = true
    3. client_sk остаётся 1254
```

**Ключевое:** факты **не трогаем**. `client_sk = 1254` остаётся валидным, просто теперь измерение заполнено.

## Как это работает с SCD2

**Проблема:** если измерение ведёт историю (SCD2), inferred member может создать лишнюю версию.

**Правило Кимбалла:** пока член помечен как inferred, **все атрибуты обрабатываются как Type 1 (перезапись)**, а не Type 2 (новая строка) **на уровне ETL**.. 
После того, inferred member «достраивается» до первой реальной версии, а дальше обновление происходит по SCD2.

```text
Inferred member: client_sk=1254, client_id=C999, name='Unknown', valid_from=2026-03-15

Приходит реальный C999 с name='Иван':
    UPDATE dim_client SET name='Иван', is_inferred=false
    WHERE client_sk=1254
    -- НЕ создаём новую строку
```

**Позже, когда C999 сменит имя на 'Иван Петров':**

```text
    UPDATE dim_client SET valid_to='2026-06-01', is_current=false
    WHERE client_sk=1254
    INSERT INTO dim_client (client_sk=2000, client_id=C999, name='Иван Петров', ...)
```


## Мониторинг

Inferred members — это **временное состояние**, а не норма. Нужен мониторинг:

```sql
-- Сколько inferred members ждут данных
SELECT COUNT(*) AS pending_members
FROM dim_client
WHERE is_inferred = true;

-- Как долго ждут и сколько фактов привязано
SELECT 
    d.client_id,
    d.first_seen_at,
    DATEDIFF(hour, d.first_seen_at, NOW()) AS waiting_hours,
    COUNT(f.sale_id) AS attached_facts
FROM dim_client d
LEFT JOIN fact_sales f ON d.client_sk = f.client_sk
WHERE d.is_inferred = true
GROUP BY d.client_id, d.first_seen_at
ORDER BY d.first_seen_at;
```

**Зачем:** inferred member может «зависнуть» навсегда, если:

- Источник никогда не пришлёт данные
    
- Бизнес-ключ изменился
    
- Extract сломался

**Действие:** алерт, если inferred member старше N часов/дней.

### Поля с метками времени:

**`load_dts`** — когда строка физически загружена в DWH
**Технический аудит.** Ставится автоматически при каждом INSERT/UPDATE в таблицу.


**`first_seen_at`** — когда факт впервые сослался на этот элемент
Поле `first_seen_at` ставится, когда создаём inferred member — то есть когда **первый факт** пришёл с этим бизнес-ключом.

**Зачем:**

- Понять, как долго inferred member «висит» без реальных данных
    
- Мониторинг: `NOW() - first_seen_at > N часов` → алерт
    
- Понять, когда бизнес-ключ впервые появился в DWH


**`completed_at`** — когда inferred member достроился реальными данными
Ставится, когда inferred member обновился реальными атрибутами и `is_inferred` стал `false`.

**Зачем:**

- Понять, сколько времени inferred member ждал данных
    
- Метрика качества ETL: `completed_at - first_seen_at`
    
- Бизнес аудит: когда данные «доехали»


**Разница** `completed_at - first_seen_at` — **SLA задержки** late-arriving dimension.


## Приколы и грабли

### Inferred member для несуществующего клиента

**Проблема:** факт ссылается на `client_id = INVALID_123`, которого никогда не было.

**Что делать:** пометить как inferred, но **не ждать** реальных данных. Если источник подтверждает, что такого клиента нет — это ошибка данных, а не late-arriving.

### Множественные inferred members для одного бизнес-ключа

**Проблема:** два ETL-процесса одновременно создали inferred member для `C999`.

**Решение:** уникальный индекс на `client_id` (WHERE is_inferred = true) + обработка конфликтов.

### SCD2 + inferred member = лишние версии

**Проблема:** inferred member с `valid_from` создаёт строку. Когда приходит реальный клиент, SCD2 создаёт ещё одну строку.

**Решение:** пока `is_inferred = true`, обрабатывать как Type 1, не создавать новую версию.

### Забыли обновить факты

**Проблема:** inferred member обновили, но факты ссылаются на старый `client_sk`.

**На самом деле:** `client_sk` **не меняется** при обновлении inferred member. Факты остаются валидными.
### Inferred member без бизнес-ключа

**Проблема:** нельзя создать inferred member, если нет бизнес-ключа (NULL в источнике).

**Решение:** это случай для **Unknown member** (-1), не для inferred.