Используем таблицу students из предыдущего ДЗ.\
Добавим в неё больше полей и данных для наглядности.
```sql
CREATE TABLE students (
    id            INT           PRIMARY KEY,
    name          TEXT          NOT NULL,
    date_of_birth DATE          NOT NULL,
    average_grade NUMERIC(2, 1)     NULL CHECK (average_grade BETWEEN 0.0 AND 5.0)
);

INSERT INTO students (id, name, date_of_birth, average_grade)
SELECT i,
       md5(random()::TEXT),
       (CURRENT_DATE - make_interval(months => floor(random() * (30 * 12 - 20 * 12 + 1) + 20 * 12)::INT))::DATE,
       CASE WHEN random() > 0.7 THEN null
                                ELSE round((random() * (4 - 0 + 1) + 0)::NUMERIC, 1)
       END
FROM generate_series (1, 1000000) i;
```

###### 1
Создание индекса по дате рождения
```sql
CREATE INDEX ON students (date_of_birth);
```

###### 2
Поиск студентов, которые родились в 2000 году
```sql
EXPLAIN ANALYZE
SELECT *
FROM students
WHERE date_of_birth BETWEEN '2000-01-01' AND '2000-12-31';
```
План запроса
```
Bitmap Heap Scan on students  (cost=1466.51..13103.46 rows=107130 width=47) (actual time=4.169..22.723 rows=99015.00 loops=1)
  Recheck Cond: ((date_of_birth >= '2000-01-01'::date) AND (date_of_birth <= '2000-12-31'::date))
  Heap Blocks: exact=10030
  Buffers: shared hit=10117
  ->  Bitmap Index Scan on students_date_of_birth_idx  (cost=0.00..1439.72 rows=107130 width=0) (actual time=3.093..3.093 rows=99015.00 loops=1)
        Index Cond: ((date_of_birth >= '2000-01-01'::date) AND (date_of_birth <= '2000-12-31'::date))
        Index Searches: 1
        Buffers: shared hit=87
Planning Time: 0.060 ms
Execution Time: 25.634 ms
```

###### 3
Создание индекса по имени в нижнем регистре.\
Предварительно нужно установить расширение pg_trgm.
```sql
CREATE EXTENSION pg_trgm;

CREATE INDEX ON students USING GIN (lower(name) gin_trgm_ops);
```
Поиск студентов по совпадению имени.\
Нужно помнить про lower(name), иначе индекс не будет работать.
```sql
EXPLAIN ANALYZE
SELECT *
FROM students
WHERE lower(name) LIKE '%abc%';
```
План запроса
```
Bitmap Heap Scan on students  (cost=147.75..10480.78 rows=20202 width=47) (actual time=2.060..10.867 rows=7393.00 loops=1)
  Recheck Cond: (lower(name) ~~ '%abc%'::text)
  Heap Blocks: exact=5250
  Buffers: shared hit=5256
  ->  Bitmap Index Scan on students_lower_idx  (cost=0.00..142.70 rows=20202 width=0) (actual time=1.337..1.337 rows=7393.00 loops=1)
        Index Cond: (lower(name) ~~ '%abc%'::text)
        Index Searches: 1
        Buffers: shared hit=6
Planning:
  Buffers: shared hit=1
Planning Time: 0.122 ms
Execution Time: 11.230 ms
```

###### 4
Создание индекса по среднему баллу, если он заполнен
```sql
CREATE INDEX ON students (average_grade) WHERE average_grade IS NOT NULL;
```
Поиск студентов со средним баллом 5.0
```sql
EXPLAIN ANALYZE
SELECT *
FROM students
WHERE average_grade = 5.0;
```
План запроса
```
Bitmap Heap Scan on students  (cost=137.00..9908.40 rows=7300 width=47) (actual time=1.063..4.241 rows=7053.00 loops=1)
  Recheck Cond: (average_grade = 5.0)
  Heap Blocks: exact=5091
  Buffers: shared hit=5113
  ->  Bitmap Index Scan on students_average_grade_idx  (cost=0.00..135.18 rows=7300 width=0) (actual time=0.495..0.495 rows=7053.00 loops=1)
        Index Cond: (average_grade = 5.0)
        Index Searches: 1
        Buffers: shared hit=22
Planning Time: 0.072 ms
Execution Time: 4.468 ms
```

###### 5
Создание индекса по дате рождения и среднему баллу, если он заполнен
```sql
CREATE INDEX ON students (date_of_birth, average_grade) WHERE average_grade IS NOT NULL;
```
Поиск студентов, которые родились в 2000 году и средним баллом 5.0
```sql
EXPLAIN ANALYZE
SELECT *
FROM students
WHERE date_of_birth BETWEEN '2000-01-01' AND '2000-12-31'
  AND average_grade = 5.0;
```
План запроса
```
Bitmap Heap Scan on students  (cost=49.81..2456.53 rows=782 width=47) (actual time=0.235..0.885 rows=721.00 loops=1)
  Recheck Cond: ((date_of_birth >= '2000-01-01'::date) AND (date_of_birth <= '2000-12-31'::date) AND (average_grade = 5.0))
  Heap Blocks: exact=697
  Buffers: shared hit=743
  ->  Bitmap Index Scan on students_date_of_birth_average_grade_idx  (cost=0.00..49.61 rows=782 width=0) (actual time=0.171..0.171 rows=721.00 loops=1)
        Index Cond: ((date_of_birth >= '2000-01-01'::date) AND (date_of_birth <= '2000-12-31'::date) AND (average_grade = 5.0))
        Index Searches: 13
        Buffers: shared hit=46
Planning Time: 0.090 ms
Execution Time: 0.918 ms
```
Составной индекс нужен, если поиск студентов всегда происходит по этим двум полям.\
Мне кажется, в реальности поиск будет по этим полям в отдельности, и такой индекс избыточен.

Если удалить индекс по среднему баллу, то при поиске только по среднему баллу на моих данных используется составной.\
В [документации](https://postgrespro.ru/docs/postgrespro/current/indexes-multicolumn) сказано
"наиболее эффективен он при ограничениях по ведущим (левым) столбцам".
```sql
DROP INDEX students_average_grade_idx;

EXPLAIN ANALYZE
SELECT *
FROM students
WHERE average_grade = 5.0;
```
План запроса
```
Bitmap Heap Scan on students  (cost=588.58..10359.97 rows=7300 width=47) (actual time=2.064..5.852 rows=7053.00 loops=1)
  Recheck Cond: (average_grade = 5.0)
  Heap Blocks: exact=5091
  Buffers: shared hit=5509
  ->  Bitmap Index Scan on students_date_of_birth_average_grade_idx  (cost=0.00..586.75 rows=7300 width=0) (actual time=1.357..1.357 rows=7053.00 loops=1)
        Index Cond: (average_grade = 5.0)
        Index Searches: 122
        Buffers: shared hit=418
Planning Time: 0.064 ms
Execution Time: 6.083 ms
```

###### 6-7
Комментарии и проблемы указаны в п.1-5.
