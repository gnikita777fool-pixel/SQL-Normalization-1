# 📘 Пошаговое руководство по лабораторной работе №1: Нормализация данных в БД

Ниже — полное руководство для выполнения задания в связке **VS Code → Git-Bash → Docker CLI → MySQL**.

---

## 🗂 Часть 0. Подготовка окружения

### 1. Запуск Docker-контейнера с MySQL

Откройте **Git-Bash** (или терминал VS Code) и выполните:

```bash
docker run --name mysql-lab -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 -d mysql:8.0
```

Проверьте, что контейнер запущен:
```bash
docker ps
```

### 2. Подключение к MySQL

```bash
docker exec -it mysql-lab mysql -uroot -proot
```

### 3. Включение логирования действий

Чтобы вести **лог всех ваших команд**, используйте команду `tee` (в Git-Bash) или `script`. Внутри mysql-сессии включите лог SQL-команд:

```sql
-- Внутри mysql-клиента:
tee /tmp/actions.log;
```

Теперь всё, что вы вводите и видите, будет сохраняться в `/tmp/actions.log`.  
По окончании работы:
```sql
notee;
```

После выхода из контейнера скопируйте лог на хост:
```bash
docker cp mysql-lab:/tmp/actions.log ./actions.log
```

---

## 🗂 Часть 1. Создание БД и «плохой» таблицы

```sql
CREATE DATABASE sales_norm;
USE sales_norm;

CREATE TABLE sales_unnormalized_2 (
     report_id     INT,
     report_date   DATE,
     customer_id   INT,
     customer_name VARCHAR(100),
     product_id    INT,
     product_name  VARCHAR(100),
     price         DECIMAL(10,2),
     quantity      INT
);

INSERT INTO sales_unnormalized_2 VALUES
 (1101, '2026-03-05', 12, 'Дания',   101, 'Дыни',      800, 1100),
 (1101, '2026-03-05', 12, 'Дания',   102, 'Клубника',  150,  300),
 (1102, '2026-03-07', 23, 'Испания', 103, 'Яблоки',    120, 1700),
 (1103, '2026-03-08', 25, 'Албания', 104, 'Лимоны',    200,  500),
 (1104, '2026-03-10', 12, 'Дания',   101, 'Дыни',      800, 2500),
 (1105, '2026-03-12', 25, 'Албания', 103, 'Яблоки',    120, 2000),
 (1105, '2026-03-12', 25, 'Албания', 104, 'Лимоны',    200,  700);
```
<img width="760" height="694" alt="image" src="https://github.com/user-attachments/assets/3e47f173-7e16-48e4-9475-ace2d94618e2" />

Проверка:
```sql
SELECT * FROM sales_unnormalized_2;
```

---

## 🗂 Часть 2. Демонстрация аномалий

### 🔴 Аномалия обновления
Например, «Дания» решила переименоваться. Нужно править **все** её строки:
```sql
UPDATE sales_unnormalized_2
SET customer_name = 'Kingdom of Denmark'
WHERE customer_id = 12;
-- Затронуто 3 строки вместо одной!
```
<img width="506" height="124" alt="image" src="https://github.com/user-attachments/assets/188f3ac4-610b-4ff1-ab89-10a19769b90e" />

### 🔴 Аномалия вставки
Нельзя добавить нового покупателя (например, `30, 'Финляндия'`), пока у него нет отчёта — т.к. `report_id` и `product_id` — часть PK, они не могут быть NULL.
```sql
-- ОШИБКА: Column 'report_id' cannot be null
INSERT INTO sales_unnormalized_2 (report_id, report_date, customer_id, customer_name, product_id, product_name, price, quantity)
VALUES (NULL, NULL, 30, 'Финляндия', NULL, NULL, NULL, NULL);
```
<img width="769" height="98" alt="image" src="https://github.com/user-attachments/assets/7bb51342-86a4-4006-9eac-426260541147" />

### 🔴 Аномалия удаления
Если удалить все отчёты Испании — исчезнет и информация о самом покупателе:
```sql
DELETE FROM sales_unnormalized_2 WHERE customer_id = 23;
-- Вместе с отчётом 1102 теряем покупателя "Испания"
```
<img width="529" height="71" alt="image" src="https://github.com/user-attachments/assets/7bf27240-8c14-4fb1-a798-70f308cbfd8e" />

---

## 🗂 Часть 3. Дамп «до» нормализации

Выйти из mysql (`exit`) и в Git-Bash:
```bash
docker exec mysql-lab mysqldump -uroot -proot sales_norm > dump_before.sql
```

---

## 🗂 Часть 4. Нормализация — создание 4 таблиц

Снова заходим в mysql:
```bash
docker exec -it mysql-lab mysql -uroot -proot
USE sales_norm;
tee /tmp/actions.log;   -- если ещё не включён
```

```sql
-- 1. Справочник покупателей
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL
);

-- 2. Справочник товаров
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price        DECIMAL(10,2) NOT NULL
);

-- 3. Шапки отчётов
CREATE TABLE reports (
    report_id   INT PRIMARY KEY,
    report_date DATE NOT NULL,
    customer_id INT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- 4. Позиции отчётов
CREATE TABLE report_items (
    report_id  INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    PRIMARY KEY (report_id, product_id),
    FOREIGN KEY (report_id)  REFERENCES reports(report_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```
<img width="669" height="640" alt="image" src="https://github.com/user-attachments/assets/b7f957a0-bdf9-455d-9a0b-ff59731f7985" />

---

## 🗂 Часть 5. Перенос данных через `INSERT ... SELECT`

```sql
-- Покупатели
INSERT INTO customers
SELECT DISTINCT customer_id, customer_name
FROM sales_unnormalized_2;

-- Товары
INSERT INTO products
SELECT DISTINCT product_id, product_name, price
FROM sales_unnormalized_2;

-- Отчёты
INSERT INTO reports
SELECT DISTINCT report_id, report_date, customer_id
FROM sales_unnormalized_2;

-- Позиции отчётов
INSERT INTO report_items (report_id, product_id, quantity)
SELECT report_id, product_id, quantity
FROM sales_unnormalized_2;
```
<img width="741" height="428" alt="image" src="https://github.com/user-attachments/assets/0c933e9c-59fb-42ba-b681-07bd158790d6" />

---

## 🗂 Часть 6. Проверка целостности данных

### Количество записей
```sql
SELECT 'Исходная' AS src, COUNT(*) AS cnt FROM sales_unnormalized_2
UNION ALL
SELECT 'reports',   COUNT(*) FROM reports
UNION ALL
SELECT 'customers', COUNT(*) FROM customers
UNION ALL
SELECT 'products',  COUNT(*) FROM products
UNION ALL
SELECT 'items',     COUNT(*) FROM report_items;
```
<img width="587" height="316" alt="image" src="https://github.com/user-attachments/assets/b7e6296c-c8b3-4b5e-a16d-a1f15a194965" />

Ожидаем: 7 строк в исходной и в `report_items`, 3 отчёта, 3 покупателя, 4 товара.

### Сумма продаж (до и после)
```sql
SELECT SUM(price * quantity) AS total_bad FROM sales_unnormalized_2;

SELECT SUM(p.price * ri.quantity) AS total_good
FROM report_items ri
JOIN products p ON p.product_id = ri.product_id;
```
Обе суммы должны совпасть (**2 015 500**).

### Тестовые запросы
Сколько потратил покупатель с кодом 12?
```sql
SELECT c.customer_name, SUM(p.price * ri.quantity) AS total_spent
FROM customers c
JOIN reports r     ON r.customer_id = c.customer_id
JOIN report_items ri ON ri.report_id = r.report_id
JOIN products p    ON p.product_id  = ri.product_id
WHERE c.customer_id = 12
GROUP BY c.customer_name;
```

---

## 🗂 Часть 7. Демонстрация исчезновения аномалий

✅ **Обновление** — меняем имя покупателя одной строкой:
```sql
UPDATE customers SET customer_name = 'Kingdom of Denmark' WHERE customer_id = 12;
```

✅ **Вставка** — можно добавить нового покупателя без отчёта:
```sql
INSERT INTO customers (customer_id, customer_name) VALUES (30, 'Финляндия');
```

✅ **Удаление** — удаляем отчёт, покупатель остаётся:
```sql
DELETE FROM report_items WHERE report_id = 1102;
DELETE FROM reports        WHERE report_id = 1102;
-- Покупатель "Испания" по-прежнему в таблице customers
SELECT * FROM customers WHERE customer_id = 23;
```

---

## 🗂 Часть 8. Дамп «после» нормализации

```bash
docker exec mysql-lab mysqldump -uroot -proot sales_norm > dump_after.sql
```

---

## 🗂 Часть 9. Финализация

```sql
-- Переименовываем старую таблицу (или удаляем)
RENAME TABLE sales_unnormalized_2 TO sales_unnormalized_2_backup;

notee;   -- отключаем лог
exit;
```

Копируем лог на хост:
```bash
docker cp mysql-lab:/tmp/actions.log ./actions.log
```

---
