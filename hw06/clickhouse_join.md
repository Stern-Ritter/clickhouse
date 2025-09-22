## DDL:
``` sql
CREATE DATABASE imdb;

CREATE TABLE imdb.actors
(
    id         UInt32,
    first_name String,
    last_name  String,
    gender     FixedString(1)
) ENGINE = MergeTree ORDER BY (id, first_name, last_name, gender);

CREATE TABLE imdb.genres
(
    movie_id UInt32,
    genre    String
) ENGINE = MergeTree ORDER BY (movie_id, genre);


CREATE TABLE imdb.movies
(
    id   UInt32,
    name String,
    year UInt32,
    rank Float32 DEFAULT 0
) ENGINE = MergeTree ORDER BY (id, name, year);


CREATE TABLE imdb.roles
(
    actor_id   UInt32,
    movie_id   UInt32,
    role       String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree ORDER BY (actor_id, movie_id);

-- Insert Data
INSERT INTO imdb.actors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.genres
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_genres.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.movies
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.roles(actor_id, movie_id, role)
SELECT actor_id, movie_id, role
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_roles.tsv.gz',
'TSVWithNames');
```


## Запросы:
1. Найти жанры для каждого фильма
``` sql
SELECT     
	m.id,
    m.name,
    m.year,
    m.rank,
    g.genre
FROM imdb.movies m
LEFT JOIN imdb.genres g ON m.id = g.movie_id;
```

2. Запросить все фильмы, у которых нет жанра
``` sql
SELECT     
	m.id,
    m.name,
    m.year,
    m.rank,
    g.genre
FROM imdb.movies m
FULL JOIN imdb.genres g ON m.id = g.movie_id
WHERE g.genre == '';
```

3. Объединить каждую строку из таблицы “Фильмы” с каждой строкой из таблицы “Жанры”
``` sql
SELECT     
	m.id,
    m.name,
    m.year,
    m.rank,
    g.movie_id,
    g.genre
FROM imdb.movies m
CROSS JOIN imdb.genres g;
```

4. Найти жанры для каждого фильма, НЕ используя INNER JOIN
``` sql
SELECT     
	m.id,
    m.name,
    m.year,
    m.rank,
    g.genre
FROM imdb.movies m
LEFT SEMI JOIN imdb.genres g ON m.id = g.movie_id;
```

5. Найти всех актеров и актрис, снявшихся в фильме в N году
``` sql
SELECT
    a.id actor_id,
    a.first_name actor_first_name, 
    a.last_name actor_last_name,
    a.gender actor_gender,
    m.id movie_id,
    m.name movie_name,
    m.year movie_year,
    m.rank movie_rank
FROM imdb.movies m
INNER JOIN imdb.roles r ON m.id = r.movie_id
INNER JOIN imdb.actors a ON r.actor_id = a.id
WHERE movie_year == 1992;
```

6. Запросить все фильмы, у которых нет жанра, через ANTI JOIN
``` sql
SELECT     
	m.id,
    m.name,
    m.year,
    m.rank,
    g.genre
FROM imdb.movies m
LEFT ANTI JOIN imdb.genres g ON m.id = g.movie_id;
```