## Выполнение домашнего задания по второй теме "Логический уровень PostgreSQL"

### Создаем экземпляр VM в CGP
![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/04.Autovacuum/VMCreated.PNG)

### Устанавливаем PostgreSQL 13
```
desmond@postgres-4:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```

### Настройка кластера

1. Чтобы настроить наш кластер согласно предложенному файлу, редактируем файл `postgresql.conf`:
```
desmond@postgres-4:~$ sudo vim /etc/postgresql/13/main/postgresql.conf
---
max_connections = 40
---
shared_buffers = 1GB
---
effective_cache_size = 3GB
---
maintenance_work_mem = 512MB
---
checkpoint_completion_target = 0.9
---
wal_buffers = 16MB
---
default_statistics_target = 500
---
random_page_cost = 4
---
effective_io_concurrency = 2
---
work_mem = 6553kB
---
min_wal_size = 4GB
max_wal_size = 16GB
```
2. Перезагружаем кластер, чтобы применились настройки:
```
desmond@postgres-4:~$ sudo pg_ctlcluster 13 main restart
```

### Работа с pgbench и дальнейшая настройка autovacuum

1. Подключаемся к пользователю `postgres` и инициализируем проверку `pgbench`:
```
desmond@postgres-4:~$ sudo -i -u postgres
postgres@postgres-4:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.16 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.77 s (drop tables 0.00 s, create tables 0.05 s, client-side generate 0.41 s, vacuum 0.13 s, primary keys 0.17 s).
```
2. Запускаем `pgbench` и даем отработать до конца:
```
postgres@postgres-4:~$ pgbench -c8 -P 60 -T 3600 -U postgres postgres
starting vacuum...end.
---
progress: 3060.0 s, 537.7 tps, lat 14.828 ms stddev 12.661
progress: 3120.0 s, 501.4 tps, lat 15.900 ms stddev 15.392
progress: 3180.0 s, 532.9 tps, lat 14.963 ms stddev 13.542
progress: 3240.0 s, 489.9 tps, lat 16.286 ms stddev 16.345
progress: 3300.0 s, 415.3 tps, lat 19.211 ms stddev 19.483
progress: 3360.0 s, 458.8 tps, lat 17.384 ms stddev 16.829
progress: 3420.0 s, 489.1 tps, lat 16.303 ms stddev 15.338
progress: 3480.0 s, 492.1 tps, lat 16.211 ms stddev 15.476
progress: 3540.0 s, 519.3 tps, lat 15.357 ms stddev 14.803
progress: 3600.0 s, 439.0 tps, lat 18.162 ms stddev 18.937
---
```
Последние 10 минут можно отобразить в виде стеков:

![Source](https://github.com/apovyshev/PostgreSQL/blob/main/04.Autovacuum/Source.png)


3. Меняем параметры autovacuum:
```
desmond@postgres-4:~$ sudo vim /etc/postgresql/13/main/postgresql.conf
---
log_autovacuum_min_duration = 0
---
autovacuum_max_workers = 8
---
autovacuum_naptime = 10s
---
autovacuum_vacuum_threshold = 25
---
autovacuum_analyze_threshold = 25
---
autovacuum_vacuum_scale_factor = 0.01
---
autovacuum_vacuum_insert_scale_factor = 0.01
---
autovacuum_analyze_scale_factor = 0.01
---
autovacuum_vacuum_cost_delay = 10ms
---
autovacuum_vacuum_cost_limit = 4800
---

postgres@postgres-4:~$ pgbench -c8 -P 60 -T 3600 -U postgres postgres
starting vacuum...end.
---
progress: 3060.0 s, 402.6 tps, lat 19.797 ms stddev 28.289
progress: 3120.0 s, 412.5 tps, lat 19.287 ms stddev 27.762
progress: 3180.0 s, 406.6 tps, lat 19.606 ms stddev 27.630
progress: 3240.0 s, 415.0 tps, lat 19.205 ms stddev 27.253
progress: 3300.0 s, 410.4 tps, lat 19.406 ms stddev 28.410
progress: 3360.0 s, 404.3 tps, lat 19.705 ms stddev 27.944
progress: 3420.0 s, 416.5 tps, lat 19.142 ms stddev 27.532
progress: 3480.0 s, 414.5 tps, lat 19.213 ms stddev 27.628
progress: 3540.0 s, 402.3 tps, lat 19.795 ms stddev 28.200
progress: 3600.0 s, 411.3 tps, lat 19.346 ms stddev 27.447
---
```
В виде стеков последние 10 минут выглядят следующим образом:

![Configured](https://github.com/apovyshev/PostgreSQL/blob/main/04.Autovacuum/Configured.png)

Для сравнения, настройка агрессивного autovacuum дала весомый результат по сравнению с дефолтной настройкой:

![Compare](https://github.com/apovyshev/PostgreSQL/blob/main/04.Autovacuum/Compare.PNG)
