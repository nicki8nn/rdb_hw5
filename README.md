# ДЗ 5: CTE-pipeline, Recursive CTE, Materialized View, EXPLAIN

Домашнє завдання з реляційних баз даних на PostgreSQL 16 у Google Colab.

**Датасет:** [Olist Brazilian E-commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — близько 100 тис. замовлень на бразильській e-commerce-платформі Olist за 2016–2018 роки. Використано таблиці `customers`, `orders`, `order_items`, `products`, `sellers`.

**Ноутбук:** `rdb_hw5_medvecka.ipynb`

---

## Зміст

| Завдання | Що зроблено |
|---|---|
| 1. Setup і typed-схема | Raw-завантаження CSV через pandas, typed-таблиці з PRIMARY KEY, FOREIGN KEY, CHECK, перенесення через `INSERT ... SELECT`, smoke-test raw vs typed |
| 2. CTE-chain pipeline | 9 named stages (0, 0b, 1–7): RFM, `cohort_quarter`, rolling AOV 30d, `cancellation_rate`, `preferred_category` |
| 3. Recursive CTE | Синтетична таблиця `hw5_referrals`, обхід referral-tree з `depth`, `path`, захистом від циклів і depth guard |
| 4. Materialized View | `mv_olist_customer_features` на основі повного pipeline, `UNIQUE INDEX`, `REFRESH MATERIALIZED VIEW CONCURRENTLY` |
| 5. EXPLAIN ANALYZE | `EXPLAIN (ANALYZE, BUFFERS)` для повного переобчислення pipeline, lookup із MV і top-N із MV |

---

## Завдання 1. Smoke-test

Дані з CSV-файлів Olist завантажено в raw-таблиці, а потім перенесено в typed-таблиці з ключами, FOREIGN KEY та CHECK-обмеженнями. Кількість рядків у typed-таблицях повністю збігається з raw:

| Таблиця | Рядків (raw = typed) |
|---|---|
| customers | 99 441 |
| orders | 99 441 |
| order_items | 112 650 |
| products | 32 951 |
| sellers | 3 095 |

**Чому втрат немає і коли вони могли б бути.** Якби рядок порушив PRIMARY KEY, FOREIGN KEY або CHECK, PostgreSQL зупинив би весь `INSERT` з помилкою, а не відкинув би рядок мовчки. Менша кількість рядків у typed-таблиці можлива лише при свідомій фільтрації (`WHERE`, `ON CONFLICT DO NOTHING`, `DISTINCT`), якої тут немає.

> У файлі Olist колонки `product_name_lenght` і `product_description_lenght` написані з помилкою. У typed-схемі вони нормалізовані до `product_name_length` і `product_description_length`.

Stub-схему не використовувала, оскільки датасет Olist успішно завантажено. Усі завдання виконано на повному наборі даних.

---

## Ключові рішення

### Чому pipeline побудовано на `customer_unique_id`, а не на `customer_id`

В Olist `customer_id` створюється заново для кожного замовлення. Перевірка:

| Показник | Значення |
|---|---|
| Кількість `customer_id` | 99 441 |
| Кількість унікальних клієнтів (`customer_unique_id`) | 96 096 |
| Максимум замовлень на один `customer_id` | 1 |
| Максимум замовлень на одного реального клієнта | 17 |

Якщо групувати за `customer_id`, `frequency` завжди дорівнюватиме 1, а `cancellation_rate` лише 0 або 1. Тому pipeline (Завдання 2) і MV (Завдання 4) побудовано на рівні `customer_unique_id`.

### Чому `UNIQUE INDEX` на `customer_unique_id`, а не на `customer_id`

MV містить один рядок на реального клієнта, тому саме `customer_unique_id` є унікальним ключем у MV. Індекс виконує ту саму роль, що вимагає завдання: однозначно ідентифікує кожен рядок MV для `REFRESH MATERIALIZED VIEW CONCURRENTLY` і забезпечує `Index Scan` при lookup клієнта (0.029 ms у Завданні 5).

### Чому довгі запити виконано через `engine`, а не `%%sql`

jupysql під час розбору `%%sql`-клітинки обрізав довгий CTE-ланцюжок, і PostgreSQL отримував неповний SQL (`syntax error at end of input`). Тому створення MV і всі `EXPLAIN` виконано через SQLAlchemy (`engine`), який передає SQL у PostgreSQL без змін. Короткі запити залишено в `%%sql`.

### Чому індекс на `monetary` створено як `DESC NULLS LAST`

Запит top-N сортує `ORDER BY monetary DESC NULLS LAST`. Індекс `(monetary DESC)` за замовчуванням має порядок `NULLS FIRST` і не збігається із запитом, тому PostgreSQL виконав би повне сортування. Індекс `(monetary DESC NULLS LAST)` точно відповідає запиту, і план складається лише з `Limit` + `Index Scan`.

---

## Результати EXPLAIN (ANALYZE, BUFFERS)

| Сценарій | Execution Time | Buffers | Ключові операції |
|---|---|---|---|
| Повне переобчислення pipeline (stages 0–2, топ-10) | 1012.4 ms | 7 416 + temp-файли на диску | `Parallel Seq Scan` ×3, `Parallel Hash Join`, `Sort` (external merge), `GroupAggregate`, `HashAggregate`, `top-N heapsort` |
| Lookup із MV (весь запит) | 27.2 ms | 1 454 | `Seq Scan` + `Sort` у CTE `target_customer`, `Index Scan` у MV |
| Lookup із MV (лише крок по MV) | 0.029 ms | 4 | `Index Scan` по `UNIQUE INDEX` |
| Top-10 із MV | 0.098 ms | 13 | `Limit` + `Index Scan` по індексу на `monetary` |

Майже весь час lookup (27 з 27.2 ms) пішов на пошук тестового клієнта в `olist_customers`, де немає індексу на `customer_unique_id`. Сам пошук у MV зайняв 0.029 ms.

---

## Reflection

**1. Найдорожча операція.** За `EXPLAIN ANALYZE` повне переобчислення pipeline зайняло 1012 ms. JOIN-и самі по собі дешеві: `Parallel Hash Join` трьох таблиць завершився приблизно на 207 ms. Найдорожчим виявився Stage 1, тобто обчислення `order_total` через `GROUP BY customer_unique_id, order_id`: перед `GroupAggregate` PostgreSQL виконав `Sort` методом `external merge` з записом на диск (~6 МБ), і цей етап тривав до 735 ms, тобто приблизно половину всього часу. Далі `HashAggregate` для RFM по 96 096 клієнтах теж не вмістився у `work_mem` (21 batch, 7.4 МБ на диску). Отже, найдорожчі тут `Sort` і агрегація, а не JOIN.

**2. Кешування.** Так, `monetary` і решту RFM-фіч варто кешувати через MV: топ-10 з MV через `Index Scan` по індексу на `monetary` виконався за 0.098 ms (13 buffers) проти 1012 ms (7 416 buffers). Olist — історичний датасет, тому достатньо on-demand refresh. Для живого магазину я обрала б щодобовий `REFRESH CONCURRENTLY`: RFM і cohort не змінюються суттєво за годину.

**3. Коли краще звичайний VIEW.** Коли потрібні актуальні дані і запит дешевий, наприклад статус замовлень одного клієнта в особистому кабінеті: фільтр за ключем, без агрегації всієї бази.

**4. Termination recursive CTE.** Рекурсія зупиняється, коли recursive part не повертає нових рядків. При циклі A → B → A вона йшла б нескінченно. Захисти: `NOT next_id = ANY(path_ids)` і `depth < 50`. Мій тест із замкненим ланцюжком із 10 клієнтів зупинився рівно на depth 10. У PostgreSQL 14+ можна також використати `CYCLE`-clause.

**5. Головний trade-off.** Recompute-cost і latency: перерахунок коштує секунду і пише на диск, тоді як MV на 96 096 рядків займає мало місця, а добова затримка для ML-фіч прийнятна.

---

## Як запустити

1. Відкрити `rdb_hw5_medvecka.ipynb` у Google Colab.
2. **Runtime → Restart session and run all.**
3. Перша клітинка встановлює PostgreSQL 16 і створює базу `hw5_db`; далі датасет завантажується з Kaggle через `kagglehub`.
