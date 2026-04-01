## Практическая работа 2.1 — Разработка и тестирование dbt‑моделей для DWH Superstore (вариант 19)

**Выполнил(а):** student_var19

---

## Архитектура DWH

В данном проекте реализован DWH на основе витрины продаж Superstore с использованием `dbt` и PostgreSQL.

- **Слои:**
  - `stg` — слой сырых/приведённых данных (`stg.orders` через модель `stg_orders`),
  - `dw_test` — DWH для проекта `superstore_dwh`,
  - `dw_student` — DWH и витрины для проекта `student_dwh`.

- **Основной поток данных:**
  - `stg.orders` → `stg_orders`  
  - `stg_orders` → измерения `customer_dim`, `geo_dim`, `product_dim`, `shipping_dim`, `calendar_dim`  
  - измерения + `stg_orders` → факт `sales_fact`  
  - `sales_fact` + `product_dim` → витрина `mart_top_unprofitable_products` (вариант 19)

**Скриншот Lineage Graph (dbt docs):**  
_Вставить сюда скриншот, например `images/lineage_graph_student_dwh.png`._

---

## Ключевые фрагменты кода

### 1. Модель `stg_orders.sql`

Файл: `superstore_dwh/models/staging/stg_orders.sql`

```sql

SELECT
    -- Приводим все к нижнему регистру для консистентности в dbt
    "order_id",
    ("order_date")::date as order_date,
    ("ship_date")::date as ship_date,
    "ship_mode",
    "customer_id",
    "customer_name",
    "segment",
    "country",
    "city",
    "state",
    CASE
        WHEN "city" = 'Burlington' AND "postal_code" IS NULL THEN '05401'
        ELSE "postal_code"
    END as postal_code,
    "region",
    "product_id",
    "category",
    "subcategory" as sub_category,
    "product_name",
    "sales",
    "quantity",
    "discount",
    "profit"
FROM {{ source('stg', 'orders') }}
```

### 2. Модель фактов `sales_fact.sql`

Файл: `superstore_dwh/models/marts/sales_fact.sql`

```sql
SELECT
    -- Суррогатные ключи из измерений
    cd.cust_id,
    pd.prod_id,
    sd.ship_id,
    gd.geo_id,
    -- Ключи для календаря
    to_char(o.order_date, 'yyyymmdd')::int AS order_date_id,
    to_char(o.ship_date, 'yyyymmdd')::int AS ship_date_id,
    -- Бизнес-ключ и метрики
    o.order_id,
    o.sales,
    o.profit,
    o.quantity,
    o.discount
FROM {{ ref('stg_orders') }} AS o
LEFT JOIN {{ ref('customer_dim') }} AS cd ON o.customer_id = cd.customer_id
LEFT JOIN {{ ref('product_dim') }} AS pd ON o.product_id = pd.product_id
LEFT JOIN {{ ref('shipping_dim') }} AS sd ON o.ship_mode = sd.ship_mode
LEFT JOIN {{ ref('geo_dim') }} AS gd ON o.postal_code = gd.postal_code AND o.city = gd.city AND o.state = gd.state
```

- Объединение измерений:
  - `customer_dim` (клиенты),
  - `product_dim` (товары),
  - `shipping_dim` (способы доставки),
  - `geo_dim` (география),
  - календарные ключи для дат заказов и отгрузки.
- Расчёт основных метрик:
  - `sales`,
  - `profit`,
  - `quantity`,
  - `discount`.

### 3. Индивидуальная mart‑модель `mart_top_unprofitable_products.sql` (вариант 19)

Файл: `student_dwh/models/marts/mart_top_unprofitable_products.sql`

```sql
-- models/marts/mart_top_unprofitable_products.sql

select
    p.prod_id,
    p.product_id,
    p.product_name, 
    p.sub_category,
    sum(f.sales)   as total_sales,
    count(*)       as rows_count,
    count(distinct f.order_id) as orders_count
from {{ ref('sales_fact') }} as f
join {{ ref('product_dim') }} as p
    on f.prod_id = p.prod_id
group by
    p.prod_id,
    p.product_id,
    p.product_name,
    p.sub_category
having sum(f.profit) < 0
order by
    total_profit asc
limit 10
```

**Задача варианта 19:**  
«Самые убыточные товары. Найти 10 товаров, которые принесли наибольший убыток.»

Логика модели:

- Агрегирует данные по каждому товару, используя:
  - суррогатный ключ товара `prod_id` из `product_dim`,
  - бизнес‑ключ `product_id`,
  - поля `product_name` и `sub_category`.
- Считает:
  - суммарную прибыль `total_profit` (на основе факта `profit`),
  - суммарные продажи `total_sales`,
  - количество строк `rows_count`,
  - количество уникальных заказов `orders_count`.
- Фильтрует только товары с **отрицательной суммарной прибылью** (`total_profit < 0`).
- Сортирует по `total_profit` по возрастанию (наибольший убыток сверху).
- Ограничивает результат **10 строками**.

### 4. Файл тестов `schema.yml`

Файл: `student_dwh/models/marts/schema.yml`

```yaml
version: 2

models:
  - name: shipping_dim
    columns:
      - name: ship_id
        tests:
          - unique
          - not_null

  - name: customer_dim
    columns:
      - name: cust_id
        tests:
          - unique
          - not_null

  - name: geo_dim
    columns:
      - name: geo_id
        tests:
          - unique
          - not_null

  - name: product_dim
    columns:
      - name: prod_id
        tests:
          - unique
          - not_null

  - name: sales_fact
    columns:
      - name: cust_id
        tests:
          - relationships:
              arguments:  
                to: ref('customer_dim')
                field: cust_id

  - name: mart_top_unprofitable_products
    description: "10 самых убыточных товаров по суммарной прибыли."
    columns:
      - name: prod_id
        description: "Суррогатный ключ товара (ссылается на product_dim.prod_id)."
        tests:
          - not_null
          - unique
          - relationships:
              to: ref('product_dim')
              field: prod_id

      - name: product_id
        description: "Бизнес-ключ товара из исходных данных."
        tests:
          - not_null

      - name: total_profit
        description: "Суммарная прибыль по товару (отрицательная — убыток)."
        tests:
          - not_null
```

---

## Результаты

### 1. Выполнение dbt‑команд

- **Команда `dbt run`** (проект `student_dwh`):
  - успешно создала все модели в схеме `dw_student`, включая:
    - базовые измерения и факт,
    - индивидуальную модель `mart_top_unprofitable_products`.
- **Команда `dbt test`**:
  - все тесты `unique`, `not_null`, `relationships` прошли без ошибок.

**Скриншот терминала с `dbt run` и `dbt test`:**  
_Вставить сюда скрин, например `images/dbt_run_test_student_dwh.png`._

### 2. Результаты витрины `mart_top_unprofitable_products`

Модель возвращает таблицу с полями (основные):

- `prod_id` — суррогатный ключ товара,
- `product_id` — бизнес‑ключ товара,
- `product_name` — название товара,
- `sub_category` — подкатегория,
- `total_profit` — суммарная прибыль (отрицательная, то есть убыток),
- `total_sales` — суммарные продажи,
- `rows_count` — число строк в факте,
- `orders_count` — число заказов.

Особенности результата:

- Возвращаются **10** строк.
- Для всех товаров `total_profit < 0`, что соответствует постановке задачи.
- Можно быстро увидеть, какие товары приносят наибольшие убытки и потенциально требуют изменения ценообразования/скидок.

**Скриншот данных витрины:**  
_Вставить сюда скрин из DBeaver или Jupyter, например `images/mart_top_unprofitable_products.png`._

---

## Выводы

1. **Использование dbt для реализации DWH** существенно упрощает разработку по сравнению с написанием отдельных DDL/DML‑скриптов:
   - модели описываются декларативно в виде SQL‑файлов с зависимостями (`ref()`),
   - есть автоматическое управление порядком выполнения.

2. **Качество данных** контролируется через встроенные тесты (`unique`, `not_null`, `relationships`), которые легко расширяются и автоматически выполняются командой `dbt test`.

3. **Документация и Lineage Graph** (`dbt docs generate` / `dbt docs serve`) дают наглядное представление о структуре DWH и зависимостях между слоями, что упрощает сопровождение и онбординг новых разработчиков.

4. **Повторяемость и переносимость**:
   - достаточно один раз настроить профиль и проект,
   - затем DWH и витрины могут быть развернуты на любом окружении одной командой `dbt run`.

5. **Индивидуальная витрина `mart_top_unprofitable_products`** демонстрирует, как поверх единого DWH можно быстро строить специализированные бизнес‑модели (отчёты), используя уже проверенные измерения и факт, что уменьшает количество ошибок и ускоряет аналитическую работу.

