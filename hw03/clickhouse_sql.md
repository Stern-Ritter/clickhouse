1. Создайте новую базу данных и перейдите в неё.
``` sql
CREATE DATABASE restaurant;
USE restaurant;
```
2. Разработайте таблицу для бизнес-кейса "Меню ресторана" с минимум пятью полями. Наполните таблицу данными, используя модификаторы (например, Nullable, LowCardinality), где это необходимо. Не забудьте добавить комментарии к полям.
``` sql
CREATE TABLE menu 
(
	id UInt32 COMMENT 'Идентификатор блюда',
	name String COMMENT 'Названи блюда',
	category LowCardinality(String) COMMENT 'Категория блюда',
	price Decimal(9, 2) COMMENT 'Цена блюда в руб.',
	is_available Bool DEFAULT true COMMENT 'Флаг доступности блюда',
	weight_grams Nullable(UInt16) COMMENT 'Вес блюда в граммах',
    preparation_time  UInt8 COMMENT 'Время приготовления блюда в минутах',
    created_at DateTime DEFAULT NOW() COMMENT 'Дата добавления блюда в меню'
) ENGINE = MergeTree()
ORDER BY (category, id);

INSERT INTO menu VALUES
(1, 'Борщ', 'Супы', 350.00, true, 300, 15, DEFAULT),
(2, 'Тирамису', 'Десерты', 450.00, true, 150, 5, DEFAULT),
(3, 'Стейк', 'Горячие блюда', 1200.00, true, 400, 25, DEFAULT),
(4, 'Цезарь', 'Салаты', 420.00, false, 250, 10, DEFAULT);
```
3. Протестируйте выполнение операций CRUD на созданной таблице.
``` sql
INSERT INTO menu VALUES
(5, 'Бургер', 'Закуски', 360.00, true, 200, 15, DEFAULT);

SELECT 	
	id, 
	name, 
	category, 
	price, 
	is_available, 
	weight_grams,
	preparation_time,
	created_at, 
FROM menu
WHERE category NOT IN('Десерты', 'Супы');

ALTER TABLE menu
UPDATE name = 'Бургер c говядиной'
WHERE id = 5;
SELECT * FROM menu WHERE id = 5;


ALTER TABLE menu
DELETE 
WHERE id = 1;
SELECT * FROM menu;
```
4. Добавьте несколько новых полей в таблицу и удалите два-три существующих.
``` sql
ALTER TABLE menu
ADD COLUMN IF NOT EXISTS
	allergens Array(LowCardinality(String)) COMMENT 'Список аллергенов в блюде',
ADD COLUMN IF NOT EXISTS
	description Nullable(String) COMMENT 'Описание блюда';
SELECT * FROM menu;

ALTER TABLE menu
DROP COLUMN IF EXISTS
	is_available,
DROP COLUMN IF EXISTS
	created_at;
SELECT * FROM menu;
```
5. Выполните выборку данных (select) из любой таблицы из sample dataset
``` sql
CREATE TABLE youtube
(
    `id` String,
    `fetch_date` DateTime,
    `upload_date_str` String,
    `upload_date` Date,
    `title` String,
    `uploader_id` String,
    `uploader` String,
    `uploader_sub_count` Int64,
    `is_age_limit` Bool,
    `view_count` Int64,
    `like_count` Int64,
    `dislike_count` Int64,
    `is_crawlable` Bool,
    `has_subtitles` Bool,
    `is_ads_enabled` Bool,
    `is_comments_enabled` Bool,
    `description` String,
    `rich_metadata` Array(Tuple(call String, content String, subtitle String, title String, url String)),
    `super_titles` Array(Tuple(text String, url String)),
    `uploader_badges` String,
    `video_badges` String
)
ENGINE = MergeTree
ORDER BY (uploader, upload_date);


INSERT INTO youtube
SETTINGS input_format_null_as_default = 1
SELECT
    id,
    parseDateTimeBestEffortUSOrZero(toString(fetch_date)) AS fetch_date,
    upload_date AS upload_date_str,
    toDate(parseDateTimeBestEffortUSOrZero(upload_date::String)) AS upload_date,
    ifNull(title, '') AS title,
    uploader_id,
    ifNull(uploader, '') AS uploader,
    uploader_sub_count,
    is_age_limit,
    view_count,
    like_count,
    dislike_count,
    is_crawlable,
    has_subtitles,
    is_ads_enabled,
    is_comments_enabled,
    ifNull(description, '') AS description,
    rich_metadata,
    super_titles,
    ifNull(uploader_badges, '') AS uploader_badges,
    ifNull(video_badges, '') AS video_badges
FROM s3(
    'https://clickhouse-public-datasets.s3.amazonaws.com/youtube/original/files/*.zst',
    'JSONLines'
)
LIMIT 1_000_000;

SELECT 
    uploader, 
    COUNT(*)
FROM youtube
GROUP BY uploader;
```
6. Материализуйте выбранную таблицу, создав её копию в виде отдельной таблицы.
``` sql
CREATE TABLE youtube_uploader_stats
(
    `uploader` String,
    `upload_date` Date,
    `total_videos` AggregateFunction(count, String),
    `total_views` AggregateFunction(sum, Int64),
    `total_likes` AggregateFunction(sum, Int64),
    `total_dislikes` AggregateFunction(sum, Int64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (uploader, upload_date);

CREATE MATERIALIZED VIEW youtube_uploader_stats_mv
TO youtube_uploader_stats AS
SELECT
    uploader,
    upload_date,
    countState(id) AS total_videos,
    sumState(view_count) AS total_views,
    sumState(like_count) AS total_likes,
    sumState(dislike_count) AS total_dislikes
FROM youtube
GROUP BY uploader, upload_date;

SELECT
    uploader,
    upload_date,
    countMerge(total_videos) AS total_videos,
    sumMerge(total_views) AS total_views,
    sumMerge(total_likes) AS total_likes,
    sumMerge(total_dislikes) AS total_dislikes
FROM youtube_uploader_stats
GROUP BY uploader, upload_date
ORDER BY total_views DESC;
```
10. Попрактикуйтесь с партициями: выполните операции ATTACH, DETACH и DROP. После этого добавьте новые данные в первоначально созданную таблицу.
``` sql
SELECT 
    partition,
    name,
    active,
    rows,
    formatReadableSize(bytes_on_disk) AS size,
    modification_time
FROM system.parts 
WHERE 
    database = 'restaurant' AND 
    table = 'youtube'
ORDER BY partition;

ALTER TABLE youtube  DETACH PART 'all_1_6_1';
ALTER TABLE youtube  ATTACH PART 'all_1_6_1';
ALTER TABLE youtube  DROP PART 'all_7_7_0';
```