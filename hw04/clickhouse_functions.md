``` sql
CREATE TABLE transactions (
    transaction_id UInt32,
    user_id UInt32,
    product_id UInt32,
    quantity UInt8,
    price Float32,
    transaction_date Date
) ENGINE = MergeTree()
ORDER BY (transaction_id);

INSERT INTO transactions VALUES
(1, 101, 1001, 2, 29.99, '2023-01-15'),
(2, 102, 1002, 1, 99.50, '2023-01-16'),
(3, 103, 1003, 3, 14.95, '2023-01-17'),
(4, 104, 1004, 1, 249.00, '2023-01-18'),
(5, 105, 1005, 2, 49.99, '2023-01-19'),
(6, 101, 1006, 1, 19.95, '2023-01-20'),
(7, 106, 1007, 4, 9.99, '2023-01-21'),
(8, 107, 1008, 1, 129.00, '2023-01-22'),
(9, 108, 1009, 2, 34.50, '2023-01-23'),
(10, 109, 1010, 1, 79.99, '2023-01-24'),
(11, 110, 1011, 3, 45.00, '2023-01-25'),
(12, 111, 1012, 1, 199.99, '2023-01-26'),
(13, 112, 1013, 2, 89.95, '2023-01-27'),
(14, 113, 1014, 1, 15.50, '2023-01-28'),
(15, 114, 1015, 5, 5.99, '2023-01-29'),
(16, 115, 1016, 2, 39.99, '2023-01-30'),
(17, 116, 1017, 1, 299.00, '2023-01-31'),
(18, 117, 1018, 3, 24.99, '2023-02-01'),
(19, 118, 1019, 1, 149.95, '2023-02-02'),
(20, 119, 1020, 2, 59.00, '2023-02-03');

-- Вариант 1
-- Агрегатные функции
-- Рассчитайте общий доход от всех операций.
SELECT SUM(quantity * price) FROM transactions;

-- Найдите средний доход с одной сделки.
SELECT AVG(quantity * price) as amount FROM transactions;

-- Определите общее количество проданной продукции.
SELECT SUM(quantity) FROM transactions;

-- Подсчитайте количество уникальных пользователей, совершивших покупку.
SELECT COUNT(DISTINCT user_id) FROM transactions;



-- Функции для работы с типами данных
-- Преобразуйте `transaction_date` в строку формата `YYYY-MM-DD`.
SELECT
	transaction_date,
	formatDateTime(transaction_date, '%Y-%m-%d') as transaction_date_formatted
FROM transactions;

-- Извлеките год и месяц из `transaction_date`.
SELECT
	transaction_date,
	dateName('year', transaction_date) as year,
	dateName('month', transaction_date) as month
FROM transactions;

-- Округлите `price` до ближайшего целого числа.
SELECT 
	price,
	round(price, 0) as rounded_price
FROM transactions;

-- Преобразуйте `transaction_id` в строку.
SELECT
	transaction_id,
	toString(transaction_id) as formatted_transaction_id
FROM transactions;



-- User-Defined Functions (UDFs)
-- Создайте простую UDF для расчета общей стоимости транзакции.
CREATE OR REPLACE FUNCTION multiply_fields AS (x, y) -> round(x * y, 2);

-- Используйте созданную UDF для расчета общей цены для каждой транзакции.
SELECT 
	quantity,
	price,
	multiply_fields(quantity, price) as amount
FROM transactions;

-- Создайте UDF для классификации транзакций на «высокоценные» и «малоценные» на основе порогового значения (например, 100).
CREATE OR REPLACE FUNCTION classify_transaction AS (value) -> if(value >= 100, 'высокоценная', 'малоценная');

-- Примените UDF для категоризации каждой транзакции
SELECT 
	quantity,
	price,
	multiply_fields(quantity, price) as amount,
	classify_transaction(multiply_fields(quantity, price)) as transaction_type
FROM transactions;
```