## Выполнение домашнего задания по теме "Нагрузочное тестирование и тюнинг PostgreSQL"

### Подготовка инфраструктуры и установка sysbench

1. Череза GCP создаем ВМ в DC Франкфурта типа e2-medium с ОС Ubuntu 20.04.
2. Устанавливаем на ВМ PostreSQL 13 из пакетов собираемых postgres.org:
```
desmond@postgres-8:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
3. Выполним некоторые настройки для кластера для подключения к базе:
```
desmond@postgres-8:~$ sudo vim /etc/postgresql/13/main/postgresql.conf
---
listen_addresses = '*'
---
sudo vim /etc/postgresql/13/main/pg_hba.conf
---
# IPv4 local connections:
host    all             all             0.0.0.0/0            md5
---
desmond@postgres-8:~$ sudo pg_ctlcluster 13 main restart
```

### Подготовка БД для тестирования нагрузки
1. Создадим отдельную базу данных и роль для тестирования нагрузки:
```
desmond@postgres-8:~$ sudo -i -u postgres psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# create database configdb;
CREATE DATABASE
```

### Тестирование нагрузки

1. Перед тем как приступить к тюнингу нашей базы данных проведем первоначальное тестирование с дефолтными настройками для кластера для получения отправной точки:
```
postgres@postgres-8:~$ pgbench -i configdb
postgres@postgres-8:~$ pgbench -c8 -P 10 -T 600 -U postgres configdb
---
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 420564
latency average = 11.358 ms
latency stddev = 14.822 ms
tps = 700.907874 (including connections establishing)
tps = 700.911272 (excluding connections establishing)
---
```

По итогу у нас получилось, что tps для стандартной настройки кластера после тестирования при помощи pgbench составляет всего 700.907874.

4. Изменим параметры кластера при помощи web помощи утилиты pgtune:
```
max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```
По окончании проверки получаем следующие данные:
```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 430889
latency average = 11.088 ms
latency stddev = 13.126 ms
tps = 718.094914 (including connections establishing)
tps = 718.098277 (excluding connections establishing)
```
Благодаря утилите нам удалось добиться дополнительных +18 tps в секунду.

5. Меняем настройки на собственные:
```
shared_buffers = 2GB
maintenance_work_mem = 512MB
effective_cache_size = 3GB
random_page_cost = 1
max_wal_senders = 0
wal_level = minimal
min_wal_size = 4GB
max_wal_size = 16GB
archive_mode = off
checkpoint_completion_target = 0.9
autovacuum = off
full_page_writes = off
synchronous_commit = off
fsync = off
work_mem = 5242kB
wal_buffers = 16MB
checkpoint_timeout = 1h
```

Небольшое пояснение, почему выбраны эти параметры для изменения:

- shared_buffers - сколько выделенной памяти будет использоваться PostgreSQL для кеширования. Рекомендовано устанавливать значение в 25% от общей памяти RAM, что в нашем случае должно ровняться 1GB.

- maintenance_work_mem - задаёт максимальный объём памяти для операций обслуживания БД, в частности VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY.

- effective_cache_size - предоставляет оценку памяти, доступной для кэширования диска. Это всего лишь ориентир, а не точный объем выделенной памяти или кеша. Он не выделяет фактическую память, но сообщает оптимизатору объем кеша, доступный в ядре. Если значение этого параметра установлено слишком низким, планировщик запросов может принять решение не использовать некоторые индексы, даже если они будут полезны. Поэтому установка большого значения всегда имеет смысл.

- random_page_cost - задаёт приблизительную стоимость чтения одной произвольной страницы с диска. в нашем случае значение приближаем к seq_page_cost. В этом случае с обращением к страницам в произвольном порядке не связаны никакие дополнительные издержки, что позволит повысить скорость получения информации.

- max_wal_senders, wal_level, archive_mode - все эти параметры важны для настройки репликации, но в нашем случае мы их изменим, чтобы отключить передачу изменений через WAL в процессе загрузки. Это не только поможет сэкономить время архивации и передачи WAL, но и непосредственно ускорит некоторые команды, потому что они не записывают в WAL ничего, если в wal_level установлен уровень minimal и текущая подтранзакция (или транзакция верхнего уровня) создала и опустошила таблицу или индекс, куда затем вносятся изменения.

- max_wal_size - увеличиваем для того, чтобы отсрочить отработку контрольной точки по требованию. checkpoint дорогостоящая операция и может вызвать огромное количество операций IO.

- checkpoint_timeout, checkpoint_completion_target - увеличим время "отдыха" контрольной точки для меньшей нагрузки.

- autovacuum - отключаем автовакуум только в нашем случае для ускорения транзакций. Рекомендовано не отключать его на Prod среде.

- full_page_writes, synchronous_commit, fsync - все эти параметры отключаем исключительно для получения максимального показателя tps. Изменения не будут записаны на диск физически.

- effective_io_concurrency - задаёт допустимое число параллельных операций ввода/вывода, которое говорит PostgreSQL о том, сколько операций ввода/вывода могут быть выполнены одновременно.Диски SSD и другие виды хранилища в памяти часто могут обрабатывать множество параллельных запросов, так что оптимальным числом может быть несколько сотен, что подходит в нашем случае.


Получаем следующие данные:
```
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 613187
latency average = 7.754 ms
latency stddev = 19.833 ms
tps = 1021.897769 (including connections establishing)
tps = 1021.903000 (excluding connections establishing)
```
Получается, рискуя сохранностью данных, можно получить значимый прирост tps.