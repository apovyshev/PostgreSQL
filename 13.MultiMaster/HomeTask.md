## Выполнение домашнего задания по теме "Работа с горизонтально масштабируемым кластером"
### Вариант 2


### Подготовка инфраструктуры
Для разворачивания CockroachDB кластера будем использовать 3 виртуальные машины в GCE.
![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/13.MultiMaster/VMCreated.PNG)

### Разворачивание кластера CockroachDB

1. Для установки CockroachDB выполняем следующую команду на каждой ВМ:

<pre><details><summary>desmond@cockroachdb1:~$ wget -qO- https://binaries.cockroachdb.com/cockroach-v20.2.4.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v20.2.4.linux-amd64/cockroach /usr/local/bin/ && sudo mkdir -p /opt/cockroach && sudo chown desmond:desmond /opt/cockroach</summary>
cockroach-v20.2.4.linux-amd64/cockroach
cockroach-v20.2.4.linux-amd64/lib/libgeos.so
cockroach-v20.2.4.linux-amd64/lib/libgeos_c.so
</details></pre>

2. Далее также на каждой ВМ стартуем `cockroach`, заменяя `advertise-addr` на имя машины, на которой стартует сервис:

<pre><details><summary>desmond@cockroachdb1:~$ cockroach start --insecure --advertise-addr=cockroachdb1 --join=cockroachdb1,cockroachdb2,cockroachdb3 --cache=.25 --max-sql-memory=.25 --background</summary>
*
* WARNING: ALL SECURITY CONTROLS HAVE BEEN DISABLED!
*
* This mode is intended for non-production testing only.
*
* In this mode:
* - Your cluster is open to any client that can access any of your IP addresses.
* - Intruders with access to your machine or network can observe client-server traffic.
* - Intruders can log in without password and read or write any data in the cluster.
* - Intruders can consume all your server's resources and cause unavailability.
*
*
* INFO: To start a secure server without mandating TLS for clients,
* consider --accept-sql-without-tls instead. For other options, see:
*
* - https://go.crdb.dev/issue-v/53404/v20.2
* - https://www.cockroachlabs.com/docs/v20.2/secure-a-cluster.html
*
*
* INFO: initial startup completed
* Node will now attempt to join a running cluster, or wait for `cockroach init`.
* Client connections will be accepted after this completes successfully.
* Check the log file(s) for progress.
*
</details></pre>

3. Инициализируем кластер на любой из машин:
```
desmond@cockroachdb1:~$ cockroach init --insecure --host=cockroachdb1
Cluster successfully initialized
```

4. Кластер собран, можем попробовать подключиться к `sql`:
<pre><details><summary>desmond@cockroachdb1:~$ cockroach sql --insecure</summary>
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v20.2.4 (x86_64-unknown-linux-gnu, built 2021/01/21 00:08:24, go1.13.14) (same version as client)
# Cluster ID: 9e440256-a858-4e60-ba6e-b4f83169f0bf
No entry for terminal type "xterm-256color";
using dumb terminal settings.
#
# Enter \? for a brief introduction.
#
</details></pre>

5. Создадим базу данных и проверим наличие данных на другой машине:
```
root@:26257/defaultdb> create database taxi;
CREATE DATABASE
```
<pre><details><summary>cockroachdb2</summary>
desmond@cockroachdb2:~$ cockroach sql --insecure
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v20.2.4 (x86_64-unknown-linux-gnu, built 2021/01/21 00:08:24, go1.13.14) (same version as client)
# Cluster ID: 9e440256-a858-4e60-ba6e-b4f83169f0bf
No entry for terminal type "xterm-256color";
using dumb terminal settings.
#
# Enter \? for a brief introduction.
#
root@:26257/defaultdb> show databases;
  database_name | owner
----------------+--------
  defaultdb     | root
  postgres      | root
  system        | node
  taxi          | root
(4 rows)
</details></pre>

Как видим, база появилась и на второй ноде.

### Создание таблиц и загрузка данных в CockroachDB и PostgreSQL

1. Прежде, чем перейти к сравнению работоспособности кластера CockroachDB и PostgreSQL, для начала загрузим данные по такси Чикаго из BigQuery. Для этого экспортируем всю таблицу `taxi_trips` базф данных `chicago_taxi_trips` в уже созданный сегмент `chicago_taxi20201` в GCS. Так как выгрузить всю таблицу в один файл не получится из-за его размера, при экспорте необходимо указать * в названии файла, например, `chicago_taxi20201/ taxi_chicago_2021_*.csv`. Тогда данные автоматически разобьются на несколько частей.

![Segment](https://github.com/apovyshev/PostgreSQL/blob/main/13.MultiMaster/Segment.PNG)

Как видим, в нашем случае таблица была разбита на 292 части примерно по 250МБ каждая.

2. Создадим таблицу `taxi_trips` в CockroachDB и PostgreSQL:

<pre><details><summary>CockroachDB</summary>
root@:26257/taxi> create table taxi_trips (
axi_id text,
trip_start_timestamp TIMES               -> unique_key text,
               -> taxi_id text,
               -> trip_start_timestamp TIMESTAMP,
               -> trip_end_timestamp TIMESTAMP,
               -> trip_seconds bigint,
               -> trip_miles numeric,
               -> pickup_census_tract bigint,
               -> dropoff_census_tract bigint,
               -> pickup_community_area bigint,
               -> dropoff_community_area bigint,
               -> fare numeric,
               -> tips numeric,
               -> tolls numeric,
               -> extras numeric,
               -> trip_total numeric,
               -> payment_type text,
               -> company text,
               -> pickup_latitude numeric,
               -> pickup_longitude numeric,
               -> pickup_location text,
               -> dropoff_latitude numeric,
               -> dropoff_longitude numeric,
               -> dropoff_location text
               -> );
CREATE TABLE

Time: 46ms total (execution 45ms / network 1ms)
</details></pre>

<pre><details><summary>PostgreSQL</summary>
desmond@postgres-test:/tmp$ sudo -i -u postgres
postgres@postgres-test:~$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# \timing
Timing is on.
postgres=# create table taxi_trips (
axi_id postgres(# unique_key text,
postgres(# taxi_id text,
postgres(# trip_start_timestamp TIMESTAMP,
postgres(# trip_end_timestamp TIMESTAMP,
postgres(# trip_seconds bigint,
postgres(# trip_miles numeric,
postgres(# pickup_census_tract bigint,
postgres(# dropoff_census_tract bigint,
postgres(# pickup_community_area bigint,
postgres(# dropoff_community_area bigint,
postgres(# fare numeric,
postgres(# tips numeric,
postgres(# tolls numeric,
postgres(# extras numeric,
postgres(# trip_total numeric,
postgres(# payment_type text,
postgres(# company text,
postgres(# pickup_latitude numeric,
postgres(# pickup_longitude numeric,
postgres(# pickup_location text,
postgres(# dropoff_latitude numeric,
postgres(# dropoff_longitude numeric,
postgres(# dropoff_location text
postgres(# );
CREATE TABLE
Time: 14.753 ms
</details></pre>
По таймингу таблица в PostgreSQL создалась быстрее (46ms vs. 14.753 ms)

3. На ВМ с установленным PostgreSQL создаем отдельную папку в /tmp/ директории:
```
desmond@postgres-test:/tmp$ mkdir trips
desmond@postgres-test:/tmp$ cd trips/
```
4. Копируем данные на с сегмента GCS в папку. Количество файлов в сумме равно примерно 10ГБ:

<pre><details><summary>gsutil -m cp</summary>
desmond@postgres-test:/tmp/trips$ gsutil -m cp \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000000.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000001.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000002.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000003.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000004.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000005.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000006.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000007.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000008.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000009.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000010.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000011.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000012.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000013.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000014.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000015.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000016.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000017.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000018.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000019.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000020.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000021.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000022.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000023.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000024.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000025.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000026.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000027.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000028.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000029.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000030.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000031.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000032.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000033.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000034.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000035.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000036.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000037.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000038.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000039.csv" \
>   "gs://chicago_taxi20201/taxi_chicago_2021_000000000040.csv" \
>   .
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000000.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000001.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000002.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000003.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000004.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000005.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000006.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000007.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000008.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000009.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000010.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000011.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000012.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000013.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000014.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000015.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000016.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000017.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000018.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000019.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000020.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000023.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000021.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000022.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000024.csv...33
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000025.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000026.csv...
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000027.csv...34
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000028.csv...37
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000029.csv...:36
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000030.csv...:11
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000031.csv...:06
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000032.csv...:41
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000033.csv...:01
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000034.csv...:48
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000035.csv...:49
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000036.csv...:40
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000037.csv...:06
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000038.csv...:09
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000039.csv...:06
Copying gs://chicago_taxi20201/taxi_chicago_2021_000000000040.csv...:44
/ [41/41 files][ 10.2 GiB/ 10.2 GiB] 100% Done   8.6 MiB/s ETA 00:00:00
Operation completed over 41 objects/10.2 GiB.
</details></pre>

5. После того, как данные скачались, переходим в `psql` и копируем данные в таблицу:
<pre><details><summary>Копирование даных в таблицу</summary>
postgres@postgres-test:~$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | taxi_trips | table | postgres
(1 row)

postgres=# \timing
Timing is on.
postgres=# COPY taxi_trips(unique_key,
postgres(# taxi_id,
rip_stapostgres(# trip_start_timestamp,
postgres(# trip_end_timestamp,
postgres(# trip_seconds,
postgres(# trip_miles,
postgres(# pickup_census_tract,
postgres(# dropoff_census_tract,
postgres(# pickup_community_area,
postgres(# dropoff_community_area,
postgres(# fare,
postgres(# tips,
postgres(# tolls,
postgres(# extras,
postgres(# trip_total,
postgres(# payment_type,
postgres(# company,
postgres(# pickup_latitude,
postgres(# pickup_longitude,
postgres(# pickup_location,
postgres(# dropoff_latitude,
postgres(# dropoff_longitude,
postgres(# dropoff_location)
postgres-# FROM PROGRAM 'awk FNR-1 /tmp/trips/*.csv | cat' DELIMITER ',' CSV HEADER;
COPY 27995397
Time: 937596.686 ms (15:37.597)
</details></pre>

По времени у нас заняло примерно 15,5 минут.

6. В случае с CockroachDB копировать данные на машину не нужно, можно импортировать из GCS напрямую:
<pre><details><summary>Импорт данных</summary>
root@:26257/taxi> import into taxi_trips(unique_key,
imestamp,
trip_end_timestamp,               -> taxi_id,
               -> trip_start_timestamp,
               -> trip_end_timestamp,
               -> trip_seconds,
               -> trip_miles,
               -> pickup_census_tract,
               -> dropoff_census_tract,
               -> pickup_community_area,
               -> dropoff_community_area,
               -> fare,
               -> tips,
               -> tolls,
               -> extras,
               -> trip_total,
               -> payment_type,
               -> company,
               -> pickup_latitude,
               -> pickup_longitude,
               -> pickup_location,
               -> dropoff_latitude,
               -> dropoff_longitude,
               -> dropoff_location) CSV DATA ('gs://chicago_taxi20201/taxi_chicago_2021_000000000000.c
sv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000001.csv', 'gs://chicago_taxi20201/taxi_chicago
_2021_000000000002.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000003.csv', 'gs://chicago_ta
xi20201/taxi_chicago_2021_000000000004.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000005.cs
v', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000006.csv', 'gs://chicago_taxi20201/taxi_chicago_
2021_000000000007.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000008.csv', 'gs://chicago_tax
i20201/taxi_chicago_2021_000000000009.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000010.csv
', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000011.csv', 'gs://chicago_taxi20201/taxi_chicago_2
021_000000000012.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000013.csv', 'gs://chicago_taxi
20201/taxi_chicago_2021_000000000014.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000015.csv'
, 'gs://chicago_taxi20201/taxi_chicago_2021_000000000016.csv', 'gs://chicago_taxi20201/taxi_chicago_20
21_000000000017.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000018.csv', 'gs://chicago_taxi2
0201/taxi_chicago_2021_000000000019.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000020.csv',
 'gs://chicago_taxi20201/taxi_chicago_2021_000000000021.csv', 'gs://chicago_taxi20201/taxi_chicago_202
1_000000000022.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000023.csv', 'gs://chicago_taxi20
201/taxi_chicago_2021_000000000024.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000025.csv',
'gs://chicago_taxi20201/taxi_chicago_2021_000000000026.csv', 'gs://chicago_taxi20201/taxi_chicago_2021
_000000000027.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000028.csv', 'gs://chicago_taxi202
01/taxi_chicago_2021_000000000029.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000030.csv', '
gs://chicago_taxi20201/taxi_chicago_2021_000000000031.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_
000000000032.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000033.csv', 'gs://chicago_taxi2020
1/taxi_chicago_2021_000000000034.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000035.csv', 'g
s://chicago_taxi20201/taxi_chicago_2021_000000000036.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_0
00000000037.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000038.csv', 'gs://chicago_taxi20201
/taxi_chicago_2021_000000000039.csv', 'gs://chicago_taxi20201/taxi_chicago_2021_000000000040.csv') WIT
H DELIMITER = ',', SKIP = '1', nullif = '';
        job_id       |  status   | fraction_completed |   rows   | index_entries |    bytes
---------------------+-----------+--------------------+----------+---------------+--------------
  630346558986846209 | succeeded |                  1 | 27995398 |             0 | 10121718105
(1 row)

Time: 365.314s total (execution 365.313s / network 0.001s)
</details></pre>

Процесс импорта занял всего 6 минут.

### Тестировнаие производительности

1. Для начала сравним отработку запросов без использования индексов:
<pre><details><summary>CockroachDB</summary>
root@:26257/taxi> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(
*) as c
               -> FROM taxi_trips
               -> group by payment_type
               -> order by 3;
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Prepaid      |            0 |       40
  Way2ride     |           15 |       95
  Split        |           18 |      333
  Pcard        |            3 |     6837
  Dispute      |            0 |    13497
  Mobile       |           15 |    21571
  Prcard       |            1 |    25252
  Unknown      |            3 |    53170
  No Charge    |            6 |   153812
  Credit Card  |           17 | 12382787
  Cash         |            0 | 15338004
(11 rows)

Time: 30.074s total (execution 30.073s / network 0.001s)

root@:26257/taxi> select count(*) from taxi_trips;
   count
------------
  27995398
(1 row)

Time: 7.625s total (execution 7.624s / network 0.000s)
</details></pre>


<pre><details><summary>PostgreSQL</summary>
postgres=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(*) as c
postgres-# FROM taxi_trips
postgres-# group by payment_type
postgres-# order by 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Prepaid      |            0 |       40
 Way2ride     |           15 |       95
 Split        |           18 |      333
 Pcard        |            3 |     6837
 Dispute      |            0 |    13497
 Mobile       |           15 |    21571
 Prcard       |            1 |    25252
 Unknown      |            3 |    53170
 No Charge    |            6 |   153812
 Credit Card  |           17 | 12382786
 Cash         |            0 | 15338004
(11 rows)
</details></pre>
Как не странно, но PostgreSQL отработал быстрее.

2. Создадим индекс и проверим работу запроса еще раз:
<pre><details><summary>CockroachDB</summary>
root@:26257/taxi> create index payment_idx on taxi_trips(payment_type);
CREATE INDEX

Time: 126.279s total (execution 0.058s / network 126.221s)

root@:26257/taxi> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(
*) as c
               -> FROM taxi_trips
               -> group by payment_type
               -> order by 3;
  payment_type | tips_percent |    c
---------------+--------------+-----------
  Prepaid      |            0 |       40
  Way2ride     |           15 |       95
  Split        |           18 |      333
  Pcard        |            3 |     6837
  Dispute      |            0 |    13497
  Mobile       |           15 |    21571
  Prcard       |            1 |    25252
  Unknown      |            3 |    53170
  No Charge    |            6 |   153812
  Credit Card  |           17 | 12382787
  Cash         |            0 | 15338004
(11 rows)

Time: 22.185s total (execution 22.184s / network 0.001s)
</details></pre>

<pre><details><summary>PostgreSQL</summary>
postgres=# create index payment_idx on taxi_trips(payment_type);
CREATE INDEX
Time: 25185.926 ms (00:25.186)
postgres=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(*) as c
postgres-# FROM taxi_trips
postgres-# group by payment_type
postgres-# order by 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Prepaid      |            0 |       40
 Way2ride     |           15 |       95
 Split        |           18 |      333
 Pcard        |            3 |     6837
 Dispute      |            0 |    13497
 Mobile       |           15 |    21571
 Prcard       |            1 |    25252
 Unknown      |            3 |    53170
 No Charge    |            6 |   153812
 Credit Card  |           17 | 12382786
 Cash         |            0 | 15338004
(11 rows)

Time: 8541.642 ms (00:08.542)
</details></pre>
Обработка данных в примере лучше происходит в PostgreSQL.
