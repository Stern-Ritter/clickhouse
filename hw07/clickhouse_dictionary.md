1. DDL
``` sql
CREATE DATABASE dict;

CREATE TABLE dict.users_actions
(
    user_id	UInt64,
    action String,
    expense UInt64
) ENGINE = MergeTree ORDER BY (user_id, action);


CREATE DICTIONARY dict.users_email_dictionary (
    user_id UInt64,
    email  String
)
PRIMARY KEY user_id
SOURCE(FILE(path '/var/lib/clickhouse/user_files/users_email.csv' format 'CSV'))
LAYOUT(FLAT)
LIFETIME(300)
SETTINGS(format_csv_delimiter = ',');

INSERT INTO dict.users_actions (user_id, action, expense) VALUES
(1, 'login', 0),
(1, 'purchase', 100),
(1, 'login', 0),
(1, 'logout', 0),
(1, 'login', 0),
(1, 'purchase', 150),

(2, 'login', 0),
(2, 'purchase', 200),
(2, 'logout', 0),
(2, 'login', 0),

(3, 'login', 0),
(3, 'purchase', 75),
(3, 'logout', 0),

(4, 'login', 0),
(4, 'logout', 0),
(4, 'login', 0),
(4, 'logout', 0),

(5, 'login', 0),
(5, 'purchase', 300),
(5, 'purchase', 250),
(5, 'purchase', 180),
(5, 'logout', 0);
```

2. Запросы
    1. Тестовый запрос к словарю
    ``` sql
    SELECT dictGet('dict.users_email_dictionary', 'email', toUInt64(1));
    ```
    2.  SELECT, который возвращает: email с помощью dictGet, аккумулятивную сумму expense с окном по action, сортировку по email.
    ``` sql
    SELECT
        dictGet('dict.users_email_dictionary', 'email', user_id) AS email,
        action,
        expense,
        SUM(expense) OVER (PARTITION BY action ORDER BY user_id) AS cumulative_expense
    FROM dict.users_actions
    ORDER BY email;
    ```