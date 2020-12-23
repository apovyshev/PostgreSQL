## Выполнение домашнего задания по теме "Работа с индексами, join'ами, статистикой"

### Вариант 1

### Подготовка инфраструктуры, базы данных и таблицы

1. Разворачиваем ВМ с Ubuntu 20.04 для установки СУБД.
2. Устанавливаем PostgreSQL 13:
```
desmond@postgres-10:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
3. В GCP BigQuery берем общедоступную базу данных и таблицу. В моем случае я выбрал bigquery-public-data:austin_bikeshare.bikeshare_trips
```
Идентификатор таблицы	
bigquery-public-data:austin_bikeshare.bikeshare_trips
Размер таблицы	
129,85 МБ
Размер для длительного хранения данных	
129,85 МБ
Количество строк	
1 264 113
Время создания	
25 мая 2017 г., 06:55:42
Таблица будет удалена	
Никогда
Последнее изменение	
19 авг. 2020 г., 22:09:50
Место обработки	
US
```
2. Экспортируем в GCS.
3. На виртуальных машинах в папку `tmp` копируем наши данные:
```
desmond@postgres-10:/tmp$ gsutil -m cp -R gs://bikes_austin .
```
4. Создадим отдельную базу на обоих ВМ для загрузки данных.
```
desmond@postgres-10:/tmp$ createdb index
desmond@postgres-10:/tmp$ psql index
psql (13.1 (Ubuntu 13.1-1.pgdg18.04+1))
Type "help" for help.
```
5. В базе данных создаем точно такую же таблицу:
```
index=# create table bikes (
index(# trip_id bigint,
index(# subscriber_type text,
index(# bikeid text,
index(# start_time TIMESTAMP,
index(# start_station_id INTEGER,
index(# start_station_name text,
index(# end_station_id text,
index(# end_station_name text,
index(# duration_minutes numeric);
CREATE TABLE
```
6. При помощи `COPY` копируем из `tmp` данные в созданную таблицу:
```
index=# COPY bikes (
trip_id,
subscriber_type,
bikeid,
start_time,
start_station_id,
start_station_name,
end_station_id,
end_station_name,
duration_minutes)
FROM PROGRAM 'awk FNR-1 /tmp/bikes_austin/*.csv | cat' DELIMITER ',' CSV HEADER;
COPY 1264112
```

### Работа с индексами b-tree
1. Прежде чем создать индекс для ускорения отработки запроса выполним произвольный запрос к таблице без использования индекса и узнаем скорость отработки:
```
index=# explain analyze
index-# select subscriber_type, bikeid, start_time
index-# from bikes
index-# where end_station_id = '2712';
```
Получаем вывод:
```
 QUERY PLAN

---------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..28354.22 rows=3413 width=27) (actual time=0.430..142.128 rows=3497 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bikes  (cost=0.00..27012.92 rows=1422 width=27) (actual time=0.043..135.419 rows=1166 loops=3)
         Filter: (end_station_id = '2712'::text)
         Rows Removed by Filter: 420205
 Planning Time: 0.063 ms
 Execution Time: 142.512 ms
(8 rows)
```
Итого, запрос у нас обрабатывался 142.512 ms.

2. Создадим индекс на поле `end_station_id` и проверим скорость выполнения запрос еще раз:
```
index=# create index end_id on bikes(end_station_id);
CREATE INDEX
```
При выполнении запроса немного изменим `select`, чтобы данные не брались из кэша:
```
index=# explain analyze
index-# select subscriber_type, bikeid::numeric + 1, start_time
index-# from bikes
index-# where end_station_id = '2712';
                                                      QUERY PLAN
----------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bikes  (cost=38.88..8996.37 rows=3413 width=55) (actual time=0.248..5.556 rows=3497 loops=1)
   Recheck Cond: (end_station_id = '2712'::text)
   Heap Blocks: exact=395
   ->  Bitmap Index Scan on end_id  (cost=0.00..38.02 rows=3413 width=0) (actual time=0.165..0.165 rows=3497 loops=1)
         Index Cond: (end_station_id = '2712'::text)
 Planning Time: 0.239 ms
 Execution Time: 5.868 ms
(7 rows)
```
Как видим, запрос отработал в разы быстрее, всего 5.868 ms. В нашем случае планировщик использовал индексный поиск по битовой карте.

### Работа с индексами gin/gist

1. Прежде, чем создавать индекс на полнотекстовый поиск, запустим запрос без индекса для проверки выполнения:
```
index-# select trip_id, to_tsvector("subscriber_type"), plainto_tsquery('Kiosk')
index-# from bikes
index-# where to_tsvector("subscriber_type") @@ plainto_tsquery('Kiosk')
index-# ORDER BY trip_id DESC;
trip_id   |                          to_tsvector                           | plainto_tsquery
------------+----------------------------------------------------------------+-----------------
 9900335435 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335434 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335433 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335432 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335431 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335428 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335427 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335426 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 9900335425 | '24':1 'austin':4 'b':6 'b-cycl':5 'cycl':7 'hour':2 'kiosk':3 | 'kiosk'
 ```
Получаем вывод explain analize:
```
                                                             QUERY PLAN

------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=292836.26..293450.90 rows=5268 width=72) (actual time=4915.743..4979.529 rows=108672 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Sort  (cost=291836.23..291842.82 rows=2634 width=72) (actual time=4899.095..4910.101 rows=36224 loops=3)
         Sort Key: trip_id DESC
         Sort Method: external merge  Disk: 4576kB
         Worker 0:  Sort Method: external merge  Disk: 4504kB
         Worker 1:  Sort Method: external merge  Disk: 5408kB
         ->  Parallel Seq Scan on bikes  (cost=0.00..291686.58 rows=2634 width=72) (actual time=7.519..4857.417 rows=36224 loops=3)
               Filter: (to_tsvector(subscriber_type) @@ plainto_tsquery('Kiosk'::text))
               Rows Removed by Filter: 385147
 Planning Time: 0.112 ms
 JIT:
   Functions: 12
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 2.941 ms, Inlining 0.000 ms, Optimization 1.273 ms, Emission 16.077 ms, Total 20.291 ms
 Execution Time: 4991.811 ms
(17 rows)
```
2. Создаем индекс `gin`:
```
index=# create index text_idx on bikes using gin (to_tsvector("subscriber_type"));
ERROR:  functions in index expression must be marked IMMUTABLE
```
Создадим функцию для индекса:
```
index=# create or replace function my_to_tsvector(t text)
returns tsvector
as
$BODY$
select to_tsvector(t)::tsvector
$BODY$
language sql
immutable;
CREATE FUNCTION
```
Создаем индекс:
```
index=# create index text_idx on bikes using gin (my_to_tsvector("subscriber_type"));
CREATE INDEX
```
Проанализируем запрос:
``` 
index=# explain analyze
index-# select trip_id, my_to_tsvector("subscriber_type"), plainto_tsquery('Kiosk')
index-# from bikes
index-# where my_to_tsvector("subscriber_type") @@ plainto_tsquery('Kiosk')
index-# ORDER BY trip_id DESC;
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=20269.26..20285.06 rows=6321 width=72) (actual time=2588.397..2619.927 rows=108672 loops=1)
   Sort Key: trip_id DESC
   Sort Method: external merge  Disk: 14480kB
   ->  Bitmap Heap Scan on bikes  (cost=73.23..19870.21 rows=6321 width=72) (actual time=18.306..2494.390 rows=108672 loops=1)
         Recheck Cond: (my_to_tsvector(subscriber_type) @@ plainto_tsquery('Kiosk'::text))
         Heap Blocks: exact=6465
         ->  Bitmap Index Scan on text_idx  (cost=0.00..71.65 rows=6321 width=0) (actual time=16.501..16.502 rows=108672 loops=1)
               Index Cond: (my_to_tsvector(subscriber_type) @@ plainto_tsquery('Kiosk'::text))
 Planning Time: 0.368 ms
 Execution Time: 2630.975 ms
(10 rows)
```
 Как видим, создание индекса на функцию для полнотекстового поиска значительно уменьшило время выполнения запроса, а также индекс почти в 10 раз сократил стоимость запроса.

3. Удалим индекс `gin` и создадим с типом `gist`:
```
index=# drop index text_idx;
DROP INDEX
index=# create index text_idx_gist on bikes using gist (my_to_tsvector("subscriber_type"));
CREATE INDEX
```
Выполним анализ для сравнения индексов:
```
index=# explain analyze
index-# select trip_id, my_to_tsvector("subscriber_type"), plainto_tsquery('Kiosk')
index-# from bikes
index-# where my_to_tsvector("subscriber_type") @@ plainto_tsquery('Kiosk')
index-# ORDER BY trip_id DESC;
                                                               QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=20445.55..20461.35 rows=6321 width=72) (actual time=4889.769..4921.853 rows=108672 loops=1)
   Sort Key: trip_id DESC
   Sort Method: external merge  Disk: 14480kB
   ->  Bitmap Heap Scan on bikes  (cost=249.53..20046.50 rows=6321 width=72) (actual time=18.675..4794.820 rows=108672 loops=1)
         Recheck Cond: (my_to_tsvector(subscriber_type) @@ plainto_tsquery('Kiosk'::text))
         Heap Blocks: exact=6465
         ->  Bitmap Index Scan on text_idx_gist  (cost=0.00..247.94 rows=6321 width=0) (actual time=16.819..16.820 rows=108672 loops=1)
               Index Cond: (my_to_tsvector(subscriber_type) @@ plainto_tsquery('Kiosk'::text))
 Planning Time: 0.293 ms
 Execution Time: 4933.147 ms
(10 rows)
```
По сравнению с запросом без использования индекса, мы сильно сократили стоимость, как и в случае с `gin`, однако, время выполнения почти не изменилось.
Из офф документации:

При выборе типа индекса для использования, GiST или GIN, учитывайте следующее разница в производительности:
- Поиск по индексу GIN примерно в три раза быстрее, чем по индексу GiST
- Построение индексов GIN занимает примерно в три раза больше времени, чем GiST
- Индексы GIN обновляются умеренно медленнее, чем индексы GiST, но примерно в 10 раз медленнее, если была отключена поддержка быстрого обновления
- GIN индексы в two-to-three раз больше, чем GiST индекса

Поэтому, необходимо учитывать бизнес потребность при создании оптимального индекса.


### Работа с индексами на часть таблицы

1. Индексы также можно создавать и на часть таблицы, если нам необходимо работать только с определенными значениями. Сравним отработку запроса без индекса:
```
index=# explain analyze
index-# select trip_id, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=28055.08..28256.70 rows=1728 width=58) (actual time=184.533..189.623 rows=2260 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Sort  (cost=27055.06..27057.22 rows=864 width=58) (actual time=170.912..171.014 rows=753 loops=3)
         Sort Key: duration_minutes DESC
         Sort Method: quicksort  Memory: 126kB
         Worker 0:  Sort Method: quicksort  Memory: 128kB
         Worker 1:  Sort Method: quicksort  Memory: 101kB
         ->  Parallel Seq Scan on bikes  (cost=0.00..27012.92 rows=864 width=58) (actual time=0.516..170.047 rows=753 loops=3)
               Filter: (duration_minutes >= '1000'::numeric)
               Rows Removed by Filter: 420617
 Planning Time: 0.172 ms
 Execution Time: 189.923 ms
(13 rows)
```
2. Теперь создадим индекс на эту часть таблицы и сравним значения:
```
index=# create index duration_idx on bikes(duration_minutes) where duration_minutes >= 1000;
CREATE INDEX
```
Выведем статистику:
```
index=# explain analyze
index-# select trip_id, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=6242.50..6247.69 rows=2074 width=58) (actual time=11.847..12.073 rows=2260 loops=1)
   Sort Key: duration_minutes DESC
   Sort Method: quicksort  Memory: 378kB
   ->  Bitmap Heap Scan on bikes  (cost=47.17..6128.24 rows=2074 width=58) (actual time=0.748..9.967 rows=2260 loops=1)
         Recheck Cond: (duration_minutes >= '1000'::numeric)
         Heap Blocks: exact=1691
         ->  Bitmap Index Scan on duration_idx  (cost=0.00..46.65 rows=2074 width=0) (actual time=0.343..0.343 rows=2260 loops=1)
 Planning Time: 0.560 ms
 Execution Time: 12.319 ms
(9 rows)
```
Как видим, время выполнения и стоимость запроса существенно сократились. 

### Работа с индексами на несколько полей

1. В данном примере поэтапно рассмотрим вменя и стоимость обработки запросов без индекса, с одним индексом и с несколькими индексами для одной таблицы. Начнем с первого варианта? предварительно удалив прошлые индексы:
```
index=# explain analyze
index-# select trip_id, start_time, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where start_time < '2015-10-02'
index-# and duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=29337.69..29385.99 rows=414 width=66) (actual time=158.104..163.246 rows=268 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Sort  (cost=28337.66..28338.18 rows=207 width=66) (actual time=142.517..142.531 rows=89 loops=3)
         Sort Key: duration_minutes DESC
         Sort Method: quicksort  Memory: 40kB
         Worker 0:  Sort Method: quicksort  Memory: 36kB
         Worker 1:  Sort Method: quicksort  Memory: 34kB
         ->  Parallel Seq Scan on bikes  (cost=0.00..28329.70 rows=207 width=66) (actual time=4.513..142.334 rows=89 loops=3)
               Filter: ((start_time < '2015-10-02 00:00:00'::timestamp without time zone) AND (duration_minutes >= '1000'::numeric))
               Rows Removed by Filter: 421281
 Planning Time: 0.147 ms
 Execution Time: 163.330 ms
(13 rows)
```
2. Создадим индекс для `start_time` и сравним значения:
```
index=# create index start_idx on bikes(start_time);
CREATE INDEX
index=# explain analyze
index-# select trip_id, start_time, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where start_time < '2015-10-02'
index-# and duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=28429.82..28478.13 rows=414 width=66) (actual time=74.114..79.094 rows=268 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Sort  (cost=27429.80..27430.32 rows=207 width=66) (actual time=60.381..60.400 rows=89 loops=3)
         Sort Key: duration_minutes DESC
         Sort Method: quicksort  Memory: 39kB
         Worker 0:  Sort Method: quicksort  Memory: 34kB
         Worker 1:  Sort Method: quicksort  Memory: 38kB
         ->  Parallel Bitmap Heap Scan on bikes  (cost=5098.16..27421.84 rows=207 width=66) (actual time=10.062..60.262 rows=89 loops=3)
               Recheck Cond: (start_time < '2015-10-02 00:00:00'::timestamp without time zone)
               Filter: (duration_minutes >= '1000'::numeric)
               Rows Removed by Filter: 99102
               Heap Blocks: exact=2616
               ->  Bitmap Index Scan on start_idx  (cost=0.00..5098.04 rows=303148 width=0) (actual time=19.016..19.016 rows=297574 loops=1)
                     Index Cond: (start_time < '2015-10-02 00:00:00'::timestamp without time zone)
 Planning Time: 0.243 ms
 Execution Time: 79.157 ms
(17 rows)
```
Как видим, стоимость запроса почти не изменилась, однако выполнение сократилось в 2 раза. 

3. Создадим еще один индекс на часть таблицы, по данным из столбца `duration_minutes` и сравним новые значения:
```
index=# create index duration_idx on bikes(duration_minutes);
CREATE INDEX
index=# explain analyze
index-# select trip_id, start_time, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where start_time < '2015-10-02'
index-# and duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=6149.87..6151.11 rows=497 width=98) (actual time=3.599..3.626 rows=268 loops=1)
   Sort Key: duration_minutes DESC
   Sort Method: quicksort  Memory: 62kB
   ->  Bitmap Heap Scan on bikes  (cost=40.11..6127.61 rows=497 width=98) (actual time=1.013..3.398 rows=268 loops=1)
         Recheck Cond: (duration_minutes >= '1000'::numeric)
         Filter: (start_time < '2015-10-02 00:00:00'::timestamp without time zone)
         Rows Removed by Filter: 1992
         Heap Blocks: exact=1691
         ->  Bitmap Index Scan on duration_idx  (cost=0.00..39.98 rows=2074 width=0) (actual time=0.488..0.488 rows=2260 loops=1)
               Index Cond: (duration_minutes >= '1000'::numeric)
 Planning Time: 0.260 ms
 Execution Time: 3.698 ms

```
После создания индекса на `duration_minutes` время и стоимость упали в несколько раз. Но, если посмотреть на работу планировщика, то после создания индекса для `duration_minutes`, индекс на `start_time` уже не использовался. Удалим этот индекс и сверим значения:

```
index=# drop index start_idx;
DROP INDEX
index=# explain analyze
index-# select trip_id, start_time, start_station_name, end_station_name, duration_minutes
index-# from bikes
index-# where start_time < '2015-10-02'
index-# and duration_minutes >= 1000
index-# order by duration_minutes DESC;
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=6155.29..6156.54 rows=497 width=66) (actual time=3.146..3.173 rows=268 loops=1)
   Sort Key: duration_minutes DESC
   Sort Method: quicksort  Memory: 61kB
   ->  Bitmap Heap Scan on bikes  (cost=46.77..6133.03 rows=497 width=66) (actual time=0.816..2.964 rows=268 loops=1)
         Recheck Cond: (duration_minutes >= '1000'::numeric)
         Filter: (start_time < '2015-10-02 00:00:00'::timestamp without time zone)
         Rows Removed by Filter: 1992
         Heap Blocks: exact=1691
         ->  Bitmap Index Scan on duration_idx  (cost=0.00..46.65 rows=2074 width=0) (actual time=0.304..0.304 rows=2260 loops=1)
 Planning Time: 0.233 ms
 Execution Time: 3.216 ms
(11 rows)
```
Как видим, после удаления индекса на столбец `start_time` время и стоимость не изменилось.
Поэтому, очень важно при создании индексов пользоваться статистикой, так как индексы, которые не используются в запросах просто занимают свободное место.




