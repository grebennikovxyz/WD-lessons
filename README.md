---

# 1NF — атомарность значений

## ❌ Плохая схема (список телефонов в одной ячейке)

```sql
-- Чистим
DROP TABLE IF EXISTS bad_users CASCADE;

-- ПЛОХО: нарушаем 1NF — в phones храним список значений в одной ячейке
CREATE TABLE bad_users (
  user_id    SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  phones     TEXT                -- например: '123, 456'
);

INSERT INTO bad_users (name, phones) VALUES
  ('Alice', '123, 456'),
  ('Bob',   '789');

-- Попробуем «найти номер 456»
-- Сложные и ненадёжные LIKE-поисki, и легко словить ложные совпадения.
-- (для демонстрации проблемы)
SELECT * FROM bad_users WHERE phones LIKE '%456%';
```

## ✅ Правильная схема (атомарные значения в отдельной таблице)

```sql
-- Чистим
DROP TABLE IF EXISTS user_phones CASCADE;
DROP TABLE IF EXISTS users CASCADE;

-- НОРМАЛИЗОВАНО: телефоны — отдельные строки
CREATE TABLE users (
  user_id  SERIAL PRIMARY KEY,
  name     TEXT NOT NULL
);

CREATE TABLE user_phones (
  phone_id  SERIAL PRIMARY KEY,
  user_id   INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
  phone     TEXT NOT NULL
);

INSERT INTO users (name) VALUES ('Alice'), ('Bob');

INSERT INTO user_phones (user_id, phone) VALUES
  (1, '123'), (1, '456'),
  (2, '789');

-- Теперь простой и точный поиск
SELECT u.name, p.phone
FROM user_phones p
JOIN users u ON u.user_id = p.user_id
WHERE p.phone = '456';
```

---

# 2NF — зависимость всех неключевых полей от всего составного ключа

## ❌ Плохая схема (дублирование customer\_name и product\_name)

```sql
-- Чистим
DROP TABLE IF EXISTS bad_orders CASCADE;

-- ПЛОХО: все данные в одной таблице; названия клиента и товара дублируются
CREATE TABLE bad_orders (
  order_id      SERIAL PRIMARY KEY,
  product_id    INT NOT NULL,
  product_name  TEXT NOT NULL,
  customer_name TEXT NOT NULL
);

INSERT INTO bad_orders (product_id, product_name, customer_name) VALUES
  (10, 'Laptop', 'Alice'),
  (11, 'Phone',  'Alice'),
  (10, 'Laptop', 'Bob');       -- имя товара и клиента повторяются

-- Обновим имя товара у product_id=10 — надо менять много строк, риск рассинхрона
UPDATE bad_orders
SET product_name = 'Laptop Pro'
WHERE product_id = 10;

SELECT * FROM bad_orders ORDER BY order_id;
```

## ✅ Правильная схема (разделяем на заказы, клиентов и товары)

```sql
-- Чистим
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS customers CASCADE;
DROP TABLE IF EXISTS products CASCADE;

-- НОРМАЛИЗОВАНО
CREATE TABLE customers (
  customer_id   SERIAL PRIMARY KEY,
  customer_name TEXT NOT NULL
);

CREATE TABLE products (
  product_id   INT PRIMARY KEY,      -- показываем, что id может быть внешним
  product_name TEXT NOT NULL
);

CREATE TABLE orders (
  order_id    SERIAL PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers(customer_id),
  product_id  INT NOT NULL REFERENCES products(product_id)
);

INSERT INTO customers (customer_name) VALUES ('Alice'), ('Bob');
INSERT INTO products (product_id, product_name) VALUES
  (10, 'Laptop'), (11, 'Phone');

INSERT INTO orders (customer_id, product_id) VALUES
  (1, 10), (1, 11), (2, 10);

-- Изменение имени товара делаем один раз — без дубликатов
UPDATE products SET product_name = 'Laptop Pro' WHERE product_id = 10;

-- Просмотр заказов с именами
SELECT o.order_id, c.customer_name, p.product_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN products  p ON p.product_id  = o.product_id
ORDER BY o.order_id;
```

---

# 3NF — отсутствие транзитивных зависимостей

## ❌ Плохая схема (city и zipcode зависят друг от друга, а не от ключа строки)

```sql
-- Чистим
DROP TABLE IF EXISTS bad_orders3 CASCADE;

-- ПЛОХО: в заказе храним city и zipcode, которые логически связаны между собой
CREATE TABLE bad_orders3 (
  order_id     SERIAL PRIMARY KEY,
  customer_id  INT NOT NULL,
  city         TEXT NOT NULL,
  zipcode      TEXT NOT NULL
);

INSERT INTO bad_orders3 (customer_id, city, zipcode) VALUES
  (1, 'New York', '10001'),
  (2, 'Boston',   '02108'),
  (1, 'New York', '10001');

-- Ошибка легко вносится: опечатка в индексе/городе ломает целостность
INSERT INTO bad_orders3 (customer_id, city, zipcode) VALUES
  (3, 'New York', '10010'); -- другая комбинация: что «истинно»?

SELECT * FROM bad_orders3 ORDER BY order_id;
```

## ✅ Правильная схема (города — справочник; заказ ссылается на клиента, клиент — на город)

```sql
-- Чистим
DROP TABLE IF EXISTS orders3 CASCADE;
DROP TABLE IF EXISTS customers3 CASCADE;
DROP TABLE IF EXISTS cities CASCADE;

-- НОРМАЛИЗОВАНО: выносим city/zipcode в справочник
CREATE TABLE cities (
  city_id  SERIAL PRIMARY KEY,
  city     TEXT NOT NULL,
  zipcode  TEXT NOT NULL,
  UNIQUE (city, zipcode)
);

CREATE TABLE customers3 (
  customer_id SERIAL PRIMARY KEY,
  city_id     INT NOT NULL REFERENCES cities(city_id),
  full_name   TEXT NOT NULL
);

CREATE TABLE orders3 (
  order_id    SERIAL PRIMARY KEY,
  customer_id INT NOT NULL REFERENCES customers3(customer_id)
);

INSERT INTO cities (city, zipcode) VALUES
  ('New York', '10001'),
  ('Boston',   '02108');

INSERT INTO customers3 (city_id, full_name) VALUES
  (1, 'Alice'),
  (2, 'Bob');

INSERT INTO orders3 (customer_id) VALUES (1), (2), (1);

-- Смотрим заказы со справочником городов
SELECT o.order_id, c.full_name, ct.city, ct.zipcode
FROM orders3 o
JOIN customers3 c ON c.customer_id = o.customer_id
JOIN cities ct     ON ct.city_id    = c.city_id
ORDER BY o.order_id;

-- Меняем индекс/название города единожды в справочнике — всё согласованно
UPDATE cities SET zipcode = '10001-0001' WHERE city='New York' AND zipcode='10001';

SELECT o.order_id, c.full_name, ct.city, ct.zipcode
FROM orders3 o
JOIN customers3 c ON c.customer_id = o.customer_id
JOIN cities ct     ON ct.city_id    = c.city_id
ORDER BY o.order_id;
```

---

хочу проверить, что вы уловили разницу? Одно короткое задание по каждой нормальной форме:

* 1NF: Добавить третьий телефон Алисе в «плохую» схему и объяснить, почему поиск по номеру становится ненадёжным; затем — сделать то же на «хорошей» схеме и показать точный SELECT.
* 2NF: Переименовать товар с `product_id=11` и показать, сколько строк придётся менять в плохой схеме и в нормализованной.
* 3NF: Поменять zipcode у «New York» и показать, где меньше действий и меньше шансов на ошибку.
