## Выполнение домашнего задания по теме "Работа с журналами"

### Настройка выполнения контрольной точки
Изменить настройки частоты выполнения контрольной точки можно двумя способами: через исправление файла `postrgesql.conf` либо через изменение настроек под psql `ALTER SYSTEM`. Для удобства выберем второй вариант:
```
desmond@postgres-5:~$ sudo -i -u postgres psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

### Тестирование нагрузки при помощи pgbench

1. Создадим отдельную базу для тестирования нагрузки:
```
postgres=# CREATE DATABASE wal;
CREATE DATABASE
```
2. Выполним инициализацию `pgbench`:
```
postgres@postgres-5:~$ pgbench -i wal
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.33 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.19 s, vacuum 0.06 s, primary keys 0.08 s).
```
3. Прежде чем подать нагрузку на базу, сбросим статистику и запомним нынешнюю позицию в журнале:
```
postgres@postgres-5:~$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c wal
You are now connected to database "wal" as user "postgres".
wal=# SELECT pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------

(1 row)

wal=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/D6C1F08
(1 row)
```
4. Подаем нагрузку в течение 10 минут:
```
postgres@postgres-5:~$ pgbench -P 10 -T 600 -U postgres wal
starting vacuum...end.
---
progress: 540.0 s, 657.9 tps, lat 1.520 ms stddev 0.204
progress: 550.0 s, 657.6 tps, lat 1.520 ms stddev 0.170
progress: 560.0 s, 662.8 tps, lat 1.508 ms stddev 0.206
progress: 570.0 s, 665.4 tps, lat 1.502 ms stddev 0.222
progress: 580.0 s, 668.1 tps, lat 1.496 ms stddev 0.178
progress: 590.0 s, 653.7 tps, lat 1.529 ms stddev 0.247
progress: 600.0 s, 664.4 tps, lat 1.505 ms stddev 0.228
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 389257
latency average = 1.541 ms
latency stddev = 0.261 ms
tps = 648.760330 (including connections establishing)
tps = 648.764136 (excluding connections establishing)
```
### Просмотр статистики 

1. Измерим, какой объем журнальных файлов был сгенерирован за время выполнения теста `pgbench`, но перед этим проверим новую позицию в журнале:
```
wal=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/D6C1F08
(1 row)
wal=# SELECT pg_size_pretty('0/281361C0'::pg_lsn - '0/D6C1F08'::pg_lsn);
 pg_size_pretty
----------------
 426 MB
(1 row)
```
2. Проверим статистику, сколько было пройдено контрольных точек:
```
wal=# SELECT checkpoints_timed FROM pg_stat_bgwriter;
 checkpoints_timed
-------------------
                23
(1 row)
```
Видим, что за время тестирования нагрузки при помощи `pgbench` в среднем на 1 контрольную точку приходилось по 20MB данных.
3. Проверка общей статистики:
```
wal=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 26
checkpoints_req       | 0
checkpoint_write_time | 313874
checkpoint_sync_time  | 139
buffers_checkpoint    | 39535
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2491
buffers_backend_fsync | 0
buffers_alloc         | 2496
stats_reset           | 2020-11-25 08:44:32.897529+00
```
С полным описанием значений можно ознакомиться [здесь](https://postgrespro.ru/docs/postgrespro/12/monitoring-stats#PG-STAT-BGWRITER-VIEW).
Видим, что параметр `checkpoints_timed`, который соответствует контрольным точкам по расписанию (`checkpoint_timeout`) равен 26 и при этом для `checkpoints_req` равен 0, так как объем данных не превысил `max_wal_size`, чтобы вызвать незапланированную контрольную точку.

Если мы зададим меньшие значения для `min_wal_size` и `max_wal_size` равным `16MB` и `40MB` соответственно, сбросим статистику, подадим ту же нагрузку и заново соберем статистику, то получим следующее:
```
wal=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 1
checkpoints_req       | 49
checkpoint_write_time | 121972
checkpoint_sync_time  | 239
buffers_checkpoint    | 87633
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 2396
buffers_backend_fsync | 0
buffers_alloc         | 2370
stats_reset           | 2020-11-25 09:25:16.086154+00
```
```
wal=# SELECT pg_size_pretty('0/8E12B3C0'::pg_lsn - '0/5DDCD7A8'::pg_lsn);
 pg_size_pretty
----------------
 771 MB
(1 row)
```
Только 1 контрольная точка была запланирована, а все остальные при общем объеме нагрузки данных в 771MB были по требованию. 
Здесь можно увидеть прямую зависимость срабатывания контрольной точки от параметров `max_wal_size`.

### Cинхронный vs Асинхронный режим записи

1. По умолчани испоьзуется синхронный режим:
```
wal=# SHOW synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)
```
2. Для тестирования, запустим снова `pgbench` на несколько секунд:
```
postgres@postgres-5:~$ pgbench -P 1 -T 10 -U postgres wal
starting vacuum...end.
progress: 1.0 s, 599.0 tps, lat 1.661 ms stddev 0.210
progress: 2.0 s, 629.0 tps, lat 1.588 ms stddev 0.319
progress: 3.0 s, 640.0 tps, lat 1.562 ms stddev 0.126
progress: 4.0 s, 625.0 tps, lat 1.599 ms stddev 0.233
progress: 5.0 s, 629.0 tps, lat 1.590 ms stddev 0.366
progress: 6.0 s, 615.0 tps, lat 1.627 ms stddev 0.220
progress: 7.0 s, 621.0 tps, lat 1.608 ms stddev 0.395
progress: 8.0 s, 631.0 tps, lat 1.586 ms stddev 0.158
progress: 9.0 s, 630.0 tps, lat 1.584 ms stddev 0.189
progress: 10.0 s, 611.0 tps, lat 1.638 ms stddev 0.379
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 6231
latency average = 1.604 ms
latency stddev = 0.277 ms
tps = 623.058348 (including connections establishing)
tps = 623.328129 (excluding connections establishing)
```
3. Изменим режим записи на асинхронный и снова протестируем tps при помощи `pgbench`:
```
wal=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
wal=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
```
postgres@postgres-5:~$ pgbench -P 1 -T 10 -U postgres wal
starting vacuum...end.
progress: 1.0 s, 1322.0 tps, lat 0.753 ms stddev 0.054
progress: 2.0 s, 1351.0 tps, lat 0.740 ms stddev 0.045
progress: 3.0 s, 1370.0 tps, lat 0.730 ms stddev 0.103
progress: 4.0 s, 1365.1 tps, lat 0.732 ms stddev 0.036
progress: 5.0 s, 1302.9 tps, lat 0.767 ms stddev 0.042
progress: 6.0 s, 1296.0 tps, lat 0.771 ms stddev 0.091
progress: 7.0 s, 1308.1 tps, lat 0.765 ms stddev 0.131
progress: 8.0 s, 1355.9 tps, lat 0.737 ms stddev 0.040
progress: 9.0 s, 1363.0 tps, lat 0.734 ms stddev 0.038
progress: 10.0 s, 1359.0 tps, lat 0.735 ms stddev 0.038
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 13394
latency average = 0.746 ms
latency stddev = 0.071 ms
tps = 1339.306177 (including connections establishing)
tps = 1339.759698 (excluding connections establishing)
```
Как видим, в асинхронном режиме скорость возрасла в 2 раза.
Однако, данные режим не гарантирует сохранность всех данных при пабении сервера базы данных. Поэтому, данный режим не рекомендуется использовать при транзакциях с критичными данными.

### Контрольные суммы страниц

1. По умолчанию, ведение контрольных сумм отключено:
```
postgres=# SHOW data_checksums;
 data_checksums
----------------
 off
(1 row)
```
2. Создадим отдельный кластер с включенным ведением контрольных сумм по умолчанию:
```
desmond@postgres-5:~$ sudo pg_createcluster 13 test --  --data-checksums
desmond@postgres-5:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
13  test    5433 down   postgres /var/lib/postgresql/13/test /var/log/postgresql/postgresql-13-test.log
desmond@postgres-5:~$ sudo pg_ctlcluster 13 test start
desmond@postgres-5:~$ sudo -i -u postgres psql -p 5433
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)
```
3. Создадим таблицу и наполним ее данными:
```
postgres=# create table test_text(t text);
CREATE TABLE
postgres=# INSERT INTO test_text SELECT 'строка '||s.id FROM generate_series(1,10) AS s(id);
INSERT 0 10
postgres=# select * from test_text;
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```
4. Перед тем, как поменять несколько байт на странице, необходимо проверить, где физически в базе расположена таблица:
```
postgres=# SELECT pg_relation_filepath('test_text');
 pg_relation_filepath
----------------------
 base/13414/16384
(1 row)
```
5. Изменяем заголовок LNS для журнальной записи:
```
desmond@postgres-5:~$ sudo pg_ctlcluster 13 test stop
desmond@postgres-5:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/13/test/base/13414/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00803994 s, 1.0 kB/s
desmond@postgres-5:~$ sudo pg_ctlcluster 13 test start
```
6. Подключаемся к нашей базе и пробуем получить данные таблицы:
```
desmond@postgres-5:~$ sudo -i -u postgres psql -p 5433
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test_text;
WARNING:  page verification failed, calculated checksum 30809 but expected 57365
ERROR:  invalid page in block 0 of relation base/13414/16384
```
Ошибка говорит о том, что данные таблицы повреждены.
Чтобы цвидеть наши данные необходимо выполнить следующую команду:
```
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# select * from test_text;
WARNING:  page verification failed, calculated checksum 30809 but expected 57365
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```
Теперь, даже после того, как мы отключим `ignore_checksum_failure`, мы все равно сможем увидеть необходимые данные:
```
postgres=# SET ignore_checksum_failure = off;
SET
postgres=# select * from test_text;
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```