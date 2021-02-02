## Выполнение домашнего задания по теме "Работа с кластером высокой доступности"
### Вариант 2


### Подготовка инфраструктуры

1. Для разворачивания кластера высокой доступности при помощи расширения `pg_auto_failover` мы будем использовать отдельные хосты в GCP: хост для мониторинга, хост для master-ноды, хос для slave-ноды. Для удобства назовем их monitor-t, master-t и slave-t соответственно.

![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/12.HACluster/VMCreated.PNG) 

2. После того, как мы создали отдельные 3 виртуальные машины, на каждой из них необходимо установить PostgreSQL и само расширение `pg_auto_failover`. В нашем случае будем устанавливать PostgreSQL 13 и собирать `pg_auto_failover` из исходного кода.

### Установка PostgreSQL

1.Устанавливаем PostgreSQL, добавляем пользователя `postgres` в `sudoers` и меняем ему пароль:
```
desmond@monitor-t:~$ sudo -i
root@monitor-t:~# wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
OK
root@monitor-t:~# echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" | tee  /etc/apt/sources.list.d/pgdg.list
deb http://apt.postgresql.org/pub/repos/apt/ focal-pgdg main
root@monitor-t:~# apt update &&  apt install postgresql postgresql-contrib -y
root@monitor-t:~# usermod -a -G sudo postgres
root@monitor-t:~# passwd postgres
New password:
Retype new password:
passwd: password updated successfully
```
2. Так как мы будем создавать отдельный кластер через расширение, останавливаем сервис `postgres`:
```
root@monitor-t:~# service postgresql stop
```

### Установка `pg_auto_failover` из исходного кода

1. Клонируем репозиторий и заходим в директорию `pg_auto_failover`:
```
postgres@monitor-t:~$ git clone https://github.com/citusdata/pg_auto_failover.git
Cloning into 'pg_auto_failover'...
remote: Enumerating objects: 8324, done.
remote: Total 8324 (delta 0), reused 0 (delta 0), pack-reused 8324
Receiving objects: 100% (8324/8324), 4.99 MiB | 18.44 MiB/s, done.
Resolving deltas: 100% (6237/6237), done.
postgres@monitor-t:~$ cd pg_auto_failover/
```
2. Устанавливаем необходимые пакеты и библиотеки для компиляции:
```
postgres@monitor-t:~/pg_auto_failover$ sudo apt install make
postgres@monitor-t:~/pg_auto_failover$ sudo apt-get install postgresql-server-dev-11 libssl-dev libkrb5-dev -y
postgres@monitor-t:~/pg_auto_failover$ sudo apt-get install libssl-dev libkrb5-dev libselinux-dev libxslt-dev libxml2-dev libpam-dev libz-dev libreadline-dev postgresql-server-dev-13 libssl-dev libkrb5-dev -y
```
3. Запускаем команду `make`:
```
postgres@monitor-t:~/pg_auto_failover$ make
```
4. После компиляции запускаем установку:
```
postgres@monitor-t:~/pg_auto_failover$ sudo make install
```
5. Проверяем установку расширения:
```
postgres@monitor-t:~/pg_auto_failover$ /usr/lib/postgresql/13/bin/pg_autoctl --version
pg_autoctl version 1.4.1
pg_autoctl extension version 1.4
compiled with PostgreSQL 13.1 (Ubuntu 13.1-1.pgdg20.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0, 64-bit
compatible with Postgres 10, 11, 12, and 13
```


### Подготовка сервера мониторинга

1. Запускаем команду создания хоста мониторинга:

<pre><details><summary>postgres@monitor-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl create monitor --pgdata /var/lib/postgresql/monitor --auth trust --ssl-self-signed</summary>
08:20:20 13587 INFO  Using default --ssl-mode "require"
08:20:20 13587 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
08:20:20 13587 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
08:20:20 13587 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
08:20:20 13587 WARN  Failed to find pg_ctl command in your PATH
08:20:20 13587 INFO  Initialising a PostgreSQL cluster at "/var/lib/postgresql/monitor"
08:20:20 13587 INFO  /usr/lib/postgresql/13/bin/pg_ctl initdb -s -D /var/lib/postgresql/monitor --option '--auth=trust'
08:20:21 13587 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /var/lib/postgresql/monitor/server.crt -keyout /var/lib/postgresql/monitor/server.key -subj "/CN=monitor-t"
08:20:21 13587 INFO  Started pg_autoctl postgres service with pid 13611
08:20:21 13611 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service postgres --pgdata /var/lib/postgresql/monitor -v
08:20:21 13587 INFO  Started pg_autoctl monitor-init service with pid 13612
08:20:21 13612 INFO  Granting connection privileges on 10.128.0.45/32
08:20:21 13617 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/monitor -p 5432 -h *
08:20:21 13611 INFO  Postgres is now serving PGDATA "/var/lib/postgresql/monitor" on port 5432 with pid 13617
08:20:21 13612 WARN  NOTICE:  installing required extension "btree_gist"
08:20:21 13612 INFO  Your pg_auto_failover monitor instance is now ready on port 5432.
08:20:21 13612 INFO  Monitor has been successfully initialized.
08:20:21 13587 WARN  pg_autoctl service monitor-init exited with exit status 0
08:20:21 13611 INFO  Postgres controller service received signal SIGTERM, terminating
08:20:21 13611 INFO  Stopping pg_autoctl postgres service
08:20:21 13611 INFO  /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/monitor --wait stop --mode fast
08:20:22 13587 INFO  Stop pg_autoctl
</details></pre>

2. Зайдем в базу и проверим, что создались база `pg_auto_failover` и пользователи `autoctl` и `autoctl_node`:
```
postgres@monitor-t:/home/desmond$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.
```

<pre><details><summary>postgres=# \l</summary>
                                 List of databases
       Name       |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
------------------+----------+----------+---------+---------+-----------------------
 pg_auto_failover | autoctl  | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres         | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0        | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
                  |          |          |         |         | postgres=CTc/postgres
 template1        | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
                  |          |          |         |         | postgres=CTc/postgres
(4 rows)
</details></pre>
<pre><details><summary>postgres=# \du</summary>
                                     List of roles
  Role name   |                         Attributes                         | Member of
--------------+------------------------------------------------------------+-----------
 autoctl      |                                                            | {}
 autoctl_node |                                                            | {}
 postgres     | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 </details></pre>


3. Изменяем файл `pg_hba` для нового кластера `monitoring` для доступа к несу с наших master и slave нод:

<pre><details><summary>postgres@monitor-t:/home/desmond$ vim /var/lib/postgresql/monitor/pg_hba.conf</summary>

# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
hostssl "pg_auto_failover" "autoctl_node" 10.128.0.45/32 trust # Auto-generated by pg_auto_failover
hostssl "pg_auto_failover" "autoctl_node" 10.128.0.46/32 trust
hostssl "pg_auto_failover" "autoctl_node" 10.128.0.47/32 trust

</details></pre>

Как видим, уже одна запись была создана автоматически, добавляем наши ноды.
4. После необходимо применить изхменения. Бдуем выполнять это через терминал postgres:
```
postgres@monitor-t:/home/desmond$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# exit
```
6. Запускаем наш мониторинг:

<pre><details><summary>postgres@monitor-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl run monitor --pgdata /var/lib/postgresql/monitor --auth trust --ssl-self-signed</summary>
08:41:50 14268 INFO  Started pg_autoctl postgres service with pid 14271
08:41:50 14271 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service postgres --pgdata /var/lib/postgresql/monitor -v
08:41:50 14268 INFO  Started pg_autoctl listener service with pid 14272
08:41:50 14272 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service listener --pgdata /var/lib/postgresql/monitor -v
08:41:50 14272 INFO  Managing the monitor at postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require
08:41:50 14272 INFO  Reloaded the new configuration from "/var/lib/postgresql/.config/pg_autoctl/var/lib/postgresql/monitor/pg_autoctl.cfg"
08:41:50 14281 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/monitor -p 5432 -h *
08:41:50 14271 INFO  Postgres is now serving PGDATA "/var/lib/postgresql/monitor" on port 5432 with pid 14281
08:41:50 14272 INFO  The version of extension "pgautofailover" is "1.4" on the monitor
08:41:50 14272 INFO  Contacting the monitor to LISTEN to its events.
</details></pre>

### Установка Master и Slave нод

1. Так как у нас уже развернуты PostgreSQL 13 и расширение `pg_auto_failover` создаем ноду:

<pre><details><summary>postgres@master-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl create postgres --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require' --run --pgctl /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/master</summary>
08:47:09 13384 INFO  Using default --ssl-mode "require"
08:47:09 13384 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
08:47:09 13384 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
08:47:09 13384 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
08:47:09 13384 INFO  Started pg_autoctl postgres service with pid 13387
08:47:09 13387 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service postgres --pgdata /var/lib/postgresql/master -v
08:47:09 13384 INFO  Started pg_autoctl node-active service with pid 13388
08:47:09 13388 WARN  Failed to connect to "postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require", retrying until the server is ready
08:47:09 13388 WARN  Connection to database failed: FATAL:  no pg_hba.conf entry for host "10.128.0.46", user "autoctl_node", database "pg_auto_failover", SSL on
08:47:09 13388 WARN  Failed to connect after successful ping, please verify authentication and logs on the server at "postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require"
08:47:09 13388 WARN  Authentication might have failed on the Postgres server due to missing HBA rules.
08:55:08 13388 INFO  Successfully connected to "postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require" after 518 attempts in 479 seconds.
08:55:08 13388 INFO  Registered node 1 (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) with name "node_1" in formation "default", group 0, state "single"
08:55:08 13388 INFO  Writing keeper state file at "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/master/pg_autoctl.state"
08:55:09 13388 INFO  Writing keeper init state file at "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/master/pg_autoctl.init"
08:55:09 13388 INFO  Successfully registered as "single" to the monitor.
08:55:09 13388 INFO  FSM transition from "init" to "single": Start as a single node
08:55:09 13388 INFO  Initialising postgres as a primary
08:55:09 13388 INFO  Initialising a PostgreSQL cluster at "/var/lib/postgresql/master"
08:55:09 13388 INFO  /usr/lib/postgresql/13/bin/pg_ctl initdb -s -D /var/lib/postgresql/master --option '--auth=trust'
08:55:11 13388 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /var/lib/postgresql/master/server.crt -keyout /var/lib/postgresql/master/server.key -subj "/CN=master-t.us-central1-a.c.verdant-coyote-294709.internal"
08:55:11 13509 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/master -p 5432 -h *
08:55:11 13388 INFO  CREATE DATABASE postgres;
08:55:11 13388 INFO  The database "postgres" already exists, skipping.
08:55:11 13388 INFO  CREATE EXTENSION pg_stat_statements;
08:55:11 13388 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /var/lib/postgresql/master/server.crt -keyout /var/lib/postgresql/master/server.key -subj "/CN=master-t.us-central1-a.c.verdant-coyote-294709.internal"
08:55:11 13387 INFO  Postgres is now serving PGDATA "/var/lib/postgresql/master" on port 5432 with pid 13509
08:55:11 13388 INFO  Contents of "/var/lib/postgresql/master/postgresql-auto-failover.conf" have changed, overwriting
08:55:11 13388 WARN  Failed to resolve hostname "monitor-t" to an IP address that resolves back to the hostname on a reverse DNS lookup.
08:55:11 13388 WARN  Postgres might deny connection attempts from "monitor-t", even with the new HBA rules.
08:55:11 13388 WARN  Hint: correct setup of HBA with host names requires proper reverse DNS setup. You might want to use IP addresses.
08:55:11 13388 WARN  Using IP address "10.128.0.45" in HBA file instead of hostname "monitor-t"
08:55:11 13388 INFO  Transition complete: current state is now "single"
08:55:11 13388 INFO  keeper has been successfully initialized.
08:55:11 13388 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service node-active --pgdata /var/lib/postgresql/master -v
08:55:11 13388 INFO  Reloaded the new configuration from "/var/lib/postgresql/.config/pg_autoctl/var/lib/postgresql/master/pg_autoctl.cfg"
08:55:11 13388 INFO  pg_autoctl service is running, current state is "single"
</details></pre>
Как видим, наша нода создалась и появилась информация, что наш мастер находится в состоянии `single`

2. Проверим нашу ноду на сервере мониторинга:

<pre><details><summary>postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl show state --pgdata /var/lib/postgresql/monitor</summary>
  Name |  Node |                                                    Host:Port |       LSN | Reachable |       Current State |      Assigned State
-------+-------+--------------------------------------------------------------+-----------+-----------+---------------------+--------------------
node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/1611C58 |       yes |              single |              single
</details></pre>
Как видим, мониторинг видит нашу ноду.

3. Установим slave ноду таким же образом:

<pre><details><summary>postgres@slave-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl create postgres --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require' --run --pgctl /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/slave</summary>
09:06:28 13063 INFO  Using default --ssl-mode "require"
09:06:28 13063 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
09:06:28 13063 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
09:06:28 13063 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
09:06:28 13063 INFO  Connecting to 10.128.0.45 (port 5432)
09:06:28 13063 INFO  Using --hostname "slave-t.us-central1-a.c.verdant-coyote-294709.internal", which resolves to IP address "10.128.0.47"
09:06:28 13063 INFO  Started pg_autoctl postgres service with pid 13065
09:06:28 13065 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service postgres --pgdata /var/lib/postgresql/slave -v
09:06:28 13063 INFO  Started pg_autoctl node-active service with pid 13066
09:06:28 13066 INFO  Registered node 2 (slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432) with name "node_2" in formation "default", group 0, state "wait_standby"
09:06:28 13066 INFO  Writing keeper state file at "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.state"
09:06:28 13066 INFO  Writing keeper init state file at "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.init"
09:06:28 13066 INFO  Successfully registered as "wait_standby" to the monitor.
09:06:28 13066 INFO  FSM transition from "init" to "wait_standby": Start following a primary
09:06:28 13066 INFO  Transition complete: current state is now "wait_standby"
09:06:28 13066 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): single ➜ wait_primary
09:06:28 13066 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): wait_primary ➜ wait_primary
09:06:28 13066 INFO  Still waiting for the monitor to drive us to state "catchingup"
09:06:28 13066 WARN  Please make sure that the primary node is currently running `pg_autoctl run` and contacting the monitor.
09:06:28 13066 INFO  FSM transition from "wait_standby" to "catchingup": The primary is now ready to accept a standby
09:06:28 13066 INFO  Initialising PostgreSQL as a hot standby
09:06:28 13066 INFO   /usr/lib/postgresql/13/bin/pg_basebackup -w -d application_name=pgautofailover_standby_2 host=master-t.us-central1-a.c.verdant-coyote-294709.internal port=5432 user=pgautofailover_replicator sslmode=require --pgdata /var/lib/postgresql/backup/node_2 -U pgautofailover_replicator --verbose --progress --max-rate 100M --wal-method=stream --slot pgautofailover_standby_2
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  initiating base backup, waiting for checkpoint to complete
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  checkpoint completed
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  write-ahead log start point: 0/2000028 on timeline 1
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  starting background WAL receiver
09:06:28 13066 INFO      0/24285 kB (0%), 0/1 tablespace (...resql/backup/node_2/backup_label)
09:06:28 13066 INFO  24294/24294 kB (100%), 0/1 tablespace (.../backup/node_2/global/pg_control)
09:06:28 13066 INFO  24294/24294 kB (100%), 1/1 tablespace
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  write-ahead log end point: 0/2000100
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  waiting for background process to finish streaming ...
09:06:28 13066 INFO  pg_basebackup:
09:06:28 13066 INFO
09:06:28 13066 INFO  syncing data to disk ...
09:06:29 13066 INFO  pg_basebackup:
09:06:29 13066 INFO
09:06:29 13066 INFO  renaming backup_manifest.tmp to backup_manifest
09:06:29 13066 INFO  pg_basebackup:
09:06:29 13066 INFO
09:06:29 13066 INFO  base backup completed
09:06:29 13066 INFO  Creating the standby signal file at "/var/lib/postgresql/slave/standby.signal", and replication setup at "/var/lib/postgresql/slave/postgresql-auto-failover-standby.conf"
09:06:29 13066 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /var/lib/postgresql/slave/server.crt -keyout /var/lib/postgresql/slave/server.key -subj "/CN=slave-t.us-central1-a.c.verdant-coyote-294709.internal"
09:06:29 13066 INFO  Contents of "/var/lib/postgresql/slave/postgresql-auto-failover.conf" have changed, overwriting
09:06:29 13074 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/slave -p 5432 -h *
09:06:29 13066 INFO  PostgreSQL started on port 5432
09:06:29 13066 INFO  Fetched current list of 1 other nodes from the monitor to update HBA rules, including 1 changes.
09:06:29 13066 INFO  Ensuring HBA rules for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:06:29 13066 INFO  Transition complete: current state is now "catchingup"
09:06:29 13066 INFO  keeper has been successfully initialized.
09:06:29 13066 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service node-active --pgdata /var/lib/postgresql/slave -v
09:06:29 13065 INFO  Postgres is now serving PGDATA "/var/lib/postgresql/slave" on port 5432 with pid 13074
09:06:29 13066 INFO  Reloaded the new configuration from "/var/lib/postgresql/.config/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.cfg"
09:06:29 13066 INFO  pg_autoctl service is running, current state is "catchingup"
09:06:29 13066 INFO  Fetched current list of 1 other nodes from the monitor to update HBA rules, including 1 changes.
09:06:29 13066 INFO  Ensuring HBA rules for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:06:30 13066 INFO  Monitor assigned new state "secondary"
</details></pre>
Нода `slave` создана и приняла состояние "secondary".

4. Проверим информацию с сервера мониторинга:

<pre><details><summary>postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl show state --pgdata /var/lib/postgresql/monitor</summary>
  Name |  Node |                                                    Host:Port |       LSN | Reachable |       Current State |      Assigned State
-------+-------+--------------------------------------------------------------+-----------+-----------+---------------------+--------------------
node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/3000148 |       yes |             primary |             primary

node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/3000148 |       yes |           secondary |           secondary
</details></pre>
Наш кластер собран, хост `master-t` указан как `primary`, а `slave-t` как `secondary`.

### Проверка работоспособности

1. На нашем `primary` хосте создадим базу и таблицу: 
```
desmond@master-t:~$ sudo -i -u postgres
postgres@master-t:~$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# create database failover;
CREATE DATABASE
postgres=# \c failover;
You are now connected to database "failover" as user "postgres".
failover=# create table test(i int);
CREATE TABLE                        ^
failover=# insert into test(i) values(10000);
INSERT 0 1
```
2. Проверим данные на `secondary`:
```
desmond@slave-t:~$ sudo -i -u postgres
postgres@slave-t:~$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 failover  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)
postgres=# \c failover;
You are now connected to database "failover" as user "postgres".
failover=# select * from test;
   i
-------
 10000
(1 row)
```
Таблица с данными есть.
3. Попробуем на `secondary` внести изменения в таблицу:
```
failover=# insert into test(i) values(200);
ERROR:  cannot execute INSERT in a read-only transaction
```
Как и ожидалось, прав нет, так как это slave.

4. Попробуем переключить наши ноды, поменять местами master и slave. Выполнять команду будем с хоста мониторинга:
<pre><details><summary>postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl perform switchover --pgdata /var/lib/postgresql/monitor</summary>
09:16:28 17628 INFO  Listening monitor notifications about state changes in formation "default" and group 0
09:16:28 17628 INFO  Following table displays times when notifications are received
    Time |   Name |  Node |                                                    Host:Port |       Current State |      Assigned State
---------+--------+-------+--------------------------------------------------------------+---------------------+--------------------
09:16:28 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |             primary |            draining
09:16:28 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |           secondary |   prepare_promotion
09:16:28 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |   prepare_promotion |   prepare_promotion
09:16:28 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |   prepare_promotion |    stop_replication
09:16:28 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |             primary |      demote_timeout
09:16:28 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |            draining |      demote_timeout
09:16:28 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |      demote_timeout |      demote_timeout
09:16:29 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |    stop_replication |    stop_replication
09:16:29 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |    stop_replication |        wait_primary
09:16:29 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |      demote_timeout |             demoted
09:16:29 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |             demoted |             demoted
09:16:29 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |        wait_primary |        wait_primary
09:16:29 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |             demoted |          catchingup
09:16:45 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |          catchingup |          catchingup
09:16:46 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |          catchingup |           secondary
09:16:46 | node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |           secondary |           secondary
09:16:46 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |        wait_primary |             primary
09:16:46 | node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 |             primary |             primary
</details></pre>
```
postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl show state --pgdata /var/lib/postgresql/monitor
  Name |  Node |                                                    Host:Port |       LSN | Reachable |       Current State |      Assigned State
-------+-------+--------------------------------------------------------------+-----------+-----------+---------------------+--------------------
node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/5000110 |       yes |           secondary |           secondary
node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/5000110 |       yes |             primary |             primary
```
Наши ноды поменялись ролями.

5. Проверим редактирование данных на каждой из них:
```
# node_2
failover=# insert into test(i) values(200);
INSERT 0 1
```
```
# node_1
failover=# insert into test(i) values(100000);
FATAL:  terminating connection due to administrator command
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
failover=# insert into test(i) values(100000);
```
Наша нода `node_1` (бывший мастер) переподключилась сама и далее не разрешила внести какие-либо изменения.

### Демонстрация автоматического переключения при падении ноды.
1. Для того, чтобы продеманстрировать автоматическое переключение нод при падении мастера, создадим еще одну ноду и подключим ее к нашему кластеру:

![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/12.HACluster/VMCreated2.PNG)

2. Установим на хосте PostgreSQL и расширение `pg_auto_failover` (см. шаги выше).
3. Добавим на хост мониторинга информацию о новой ноде в `pg_hba.conf`.
4. Запустим ноду:
<pre><details><summary>postgres@slave2-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl create postgres --auth trust --ssl-self-signed --monitor 'postgres://autoctl_node@monitor-t:5432/pg_auto_failover?sslmode=require' --run --pgctl /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/slave2</summary>
09:29:16 13285 INFO  Using default --ssl-mode "require"
09:29:16 13285 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
09:29:16 13285 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
09:29:16 13285 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
09:29:16 13285 INFO  Getting nodes from the monitor for group 0 in formation "default"
09:29:16 13285 INFO  Node name on the monitor is now "node_3"
09:29:16 13285 INFO  Started pg_autoctl postgres service with pid 13290
09:29:16 13290 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service postgres --pgdata /var/lib/postgresql/slave2 -v
09:29:16 13285 INFO  Started pg_autoctl node-active service with pid 13291
09:29:16 13291 INFO  Continuing from a previous `pg_autoctl create` failed attempt
09:29:16 13291 INFO  PostgreSQL state at registration time was: PGDATA does not exists
09:29:16 13291 INFO  FSM transition from "wait_standby" to "catchingup": The primary is now ready to accept a standby
09:29:16 13291 INFO  Initialising PostgreSQL as a hot standby
09:29:16 13291 INFO   /usr/lib/postgresql/13/bin/pg_basebackup -w -d application_name=pgautofailover_standby_3 host=slave-t.us-central1-a.c.verdant-coyote-294709.internal port=5432 user=pgautofailover_replicator sslmode=require --pgdata /var/lib/postgresql/backup/node_3 -U pgautofailover_replicator --verbose --progress --max-rate 100M --wal-method=stream --slot pgautofailover_standby_3
09:29:16 13291 INFO  pg_basebackup:
09:29:16 13291 INFO
09:29:16 13291 INFO  initiating base backup, waiting for checkpoint to complete
09:29:16 13291 INFO  pg_basebackup:
09:29:16 13291 INFO
09:29:16 13291 INFO  checkpoint completed
09:29:16 13291 INFO  pg_basebackup:
09:29:16 13291 INFO
09:29:16 13291 INFO  write-ahead log start point: 0/10000028 on timeline 2
09:29:16 13291 INFO  pg_basebackup:
09:29:16 13291 INFO
09:29:16 13291 INFO  starting background WAL receiver
09:29:16 13291 INFO      0/32191 kB (0%), 0/1 tablespace (...resql/backup/node_3/backup_label)
09:29:16 13291 INFO  32200/32200 kB (100%), 0/1 tablespace (.../backup/node_3/global/pg_control)
09:29:16 13291 INFO  32200/32200 kB (100%), 1/1 tablespace
09:29:16 13291 INFO  pg_basebackup:
09:29:16 13291 INFO  write-ahead log end point: 0/10000100
09:29:16 13291 INFO  pg_basebackup: waiting for background process to finish streaming ...
09:29:16 13291 INFO  pg_basebackup: syncing data to disk ...
09:29:17 13291 INFO  pg_basebackup: renaming backup_manifest.tmp to backup_manifest
09:29:17 13291 INFO  pg_basebackup: base backup completed
09:29:17 13291 INFO  Creating the standby signal file at "/var/lib/postgresql/slave2/standby.signal", and replication setup at "/var/lib/postgresql/slave2/postgresql-auto-failover-standby.conf"
09:29:17 13291 INFO  Contents of "/var/lib/postgresql/slave2/postgresql-auto-failover-standby.conf" have changed, overwriting
09:29:17 13291 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /var/lib/postgresql/slave2/server.crt -keyout /var/lib/postgresql/slave2/server.key -subj "/CN=slave2-t.us-central1-a.c.verdant-coyote-294709.internal"
09:29:17 13291 INFO  Contents of "/var/lib/postgresql/slave2/postgresql-auto-failover.conf" have changed, overwriting
09:29:17 13298 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/slave2 -p 5432 -h *
09:29:17 13291 INFO  PostgreSQL started on port 5432
09:29:17 13290 INFO  Postgres is now serving PGDATA "/var/lib/postgresql/slave2" on port 5432 with pid 13298
09:29:17 13291 INFO  Fetched current list of 2 other nodes from the monitor to update HBA rules, including 2 changes.
09:29:17 13291 INFO  Ensuring HBA rules for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:29:17 13291 INFO  Ensuring HBA rules for node 2 "node_2" (slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:29:17 13291 INFO  Transition complete: current state is now "catchingup"
09:29:17 13291 INFO  keeper has been successfully initialized.
09:29:17 13291 INFO   /usr/lib/postgresql/13/bin/pg_autoctl do service node-active --pgdata /var/lib/postgresql/slave2 -v
09:29:17 13291 INFO  Reloaded the new configuration from "/var/lib/postgresql/.config/pg_autoctl/var/lib/postgresql/slave2/pg_autoctl.cfg"
09:29:18 13291 INFO  pg_autoctl service is running, current state is "catchingup"
09:29:18 13291 INFO  Fetched current list of 2 other nodes from the monitor to update HBA rules, including 2 changes.
09:29:18 13291 INFO  Ensuring HBA rules for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:29:18 13291 INFO  Ensuring HBA rules for node 2 "node_2" (slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:29:19 13291 INFO  Monitor assigned new state "secondary"
09:29:19 13291 INFO  FSM transition from "catchingup" to "secondary": Convinced the monitor that I'm up and running, and eligible for promotion again
09:29:19 13291 INFO  Creating replication slot "pgautofailover_standby_1"
09:29:19 13291 INFO  Creating replication slot "pgautofailover_standby_2"
09:29:19 13291 INFO  Transition complete: current state is now "secondary"
09:29:19 13291 INFO  New state for node 2 "node_2" (slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432): primary ➜ primary
09:35:57 13291 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 13291 INFO  New state for this node (node 3, "node_3") (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 13291 INFO  Monitor assigned new state "report_lsn"
09:35:57 13291 INFO  FSM transition from "secondary" to "report_lsn": Reporting the last write-ahead log location received
09:35:57 13291 INFO  Restarting standby node to disconnect replication from failed primary node, to prepare failover
09:35:57 13291 INFO  Stopping Postgres at "/var/lib/postgresql/slave2"
09:35:57 13290 INFO  Stopping pg_autoctl postgres service
09:35:57 13290 INFO  /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/slave2 --wait stop --mode fast
09:35:57 13291 INFO  Creating the standby signal file at "/var/lib/postgresql/slave2/standby.signal", and replication setup at "/var/lib/postgresql/slave2/postgresql-auto-failover-standby.conf"
09:35:57 13291 INFO  Contents of "/var/lib/postgresql/slave2/postgresql-auto-failover-standby.conf" have changed, overwriting
09:35:57 13291 INFO  Restarting Postgres at "/var/lib/postgresql/slave2"
09:35:57 14295 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/slave2 -p 5432 -h *
09:35:57 13290 WARN  PostgreSQL was not running, restarted with pid 14295
09:35:57 13291 INFO  Transition complete: current state is now "report_lsn"
09:35:57 13291 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 13291 INFO  New state for this node (node 3, "node_3") (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 13291 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ prepare_promotion
09:35:57 13291 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ prepare_promotion
09:35:57 13291 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ wait_primary
09:35:57 13291 INFO  New state for this node (node 3, "node_3") (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ join_secondary
09:35:57 13291 INFO  Monitor assigned new state "join_secondary"
09:35:57 13291 INFO  FSM transition from "report_lsn" to "join_secondary": A failover candidate has been selected, stop replication
09:35:57 13291 INFO  Preparing Postgres shutdown: CHECKPOINT;
09:35:57 13291 INFO  Stopping Postgres at "/var/lib/postgresql/slave2"
09:35:58 13290 INFO  Stopping pg_autoctl postgres service
09:35:58 13290 INFO  /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/slave2 --wait stop --mode fast
09:35:58 13291 INFO  Transition complete: current state is now "join_secondary"
09:35:58 13291 INFO  New state for this node (node 3, "node_3") (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): join_secondary ➜ join_secondary
09:36:00 13291 INFO  Monitor assigned new state "secondary"
</details></pre> 
5. Проверим наши ноды на сервере мониторинга:
```
postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl show state --pgdata /var/lib/postgresql/monitor
  Name |  Node |                                                    Host:Port |        LSN | Reachable |       Current State |      Assigned State
-------+-------+--------------------------------------------------------------+------------+-----------+---------------------+--------------------
node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/11000148 |       yes |           secondary |           secondary
node_2 |     2 |  slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/11000148 |       yes |             primary |             primary
node_3 |     3 | slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/11000148 |       yes |           secondary |           secondary
```
Наша нода подключена и используется как `secondary`.

6. Проверим содержимое таблицы на `node_3`:
```
postgres@slave2-t:/home/desmond$ psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 failover  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)

postgres=# \c failover
You are now connected to database "failover" as user "postgres".
failover=# select * from test;
   i
-------
 10000
   200
(2 rows)
```
Вся информация на месте.

7. Удалим мастер-ноду:
<pre><details><summary>postgres@slave-t:~$ /usr/lib/postgresql/13/bin/pg_autoctl drop node --pgdata /var/lib/postgresql/slave</summary>
09:35:57 17472 INFO  An instance of pg_autoctl is running with PID 13063, stopping it.
09:35:57 17472 INFO  Removing local node from the pg_auto_failover monitor.
09:35:57 17472 INFO  Removing local node state file: "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.state"
09:35:57 17472 INFO  Removing local node init state file: "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.init"
09:35:57 17472 INFO  Removed pg_autoctl node at "/var/lib/postgresql/slave" from the monitor and removed the state file "/var/lib/postgresql/.local/share/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.state"
09:35:57 17472 INFO  Stopping PostgreSQL at "/var/lib/postgresql/slave"
09:35:57 17472 INFO  /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/slave --wait stop --mode fast
09:35:57 17472 WARN  Configuration file "/var/lib/postgresql/.config/pg_autoctl/var/lib/postgresql/slave/pg_autoctl.cfg" has been preserved
09:35:57 17472 WARN  Postgres Data Directory "/var/lib/postgresql/slave" has been preserved
09:35:57 17472 INFO  drop node keeps your data and setup safe, you can still run Postgres or re-join the pg_auto_failover cluster later
09:35:57 17472 INFO  HINT: to completely remove your local Postgres instance and setup, consider `pg_autoctl drop node --destroy`
</details></pre> 

8. В этот момент на мониторинг хосте и node_1:
<pre><details><summary>Вывод мониторинга</summary>
09:35:57 14272 INFO  Setting goal state of node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to report_lsn after primary node removal.
09:35:57 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 14272 INFO  Setting goal state of node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to report_lsn after primary node removal.
09:35:57 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 14272 INFO  Removing node 2 "node_2" (slave-t.us-central1-a.c.verdant-coyote-294709.internal:5432) from formation "default" and group 0
09:35:57 14272 INFO  Setting number_sync_standbys to 0 for formation "default" now that we have 1 standby nodes set with replication-quorum.
09:35:57 14272 INFO  New state is reported by node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) with LSN 0/11000148: report_lsn
09:35:57 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 14272 INFO  Failover still in progress after 1 nodes reported their LSN and we are waiting for 1 nodes to report, activeNode is node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) and reported state "report_lsn"
09:35:57 14272 INFO  Failover still in progress after 1 nodes reported their LSN and we are waiting for 1 nodes to report, activeNode is node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) and reported state "report_lsn"
09:35:57 14272 INFO  New state is reported by node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) with LSN 0/11000148: report_lsn
09:35:57 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 14272 INFO  The current most advanced reported LSN is 0/110001C0, as reported by node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) and 1 other nodes
09:35:57 14272 INFO  Setting goal state of node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to prepare_promotion and 2 nodes reported their LSN position.
09:35:57 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ prepare_promotion
09:35:57 14272 INFO  Active node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) found failover candidate node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) being promoted (currently "report_lsn"/"prepare_promotion")
09:35:57 14272 INFO  New state is reported by node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): "prepare_promotion"
09:35:57 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ prepare_promotion
09:35:57 14272 INFO  Setting goal state of and node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to wait_primary after node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) converged to prepare_promotion.
09:35:57 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ wait_primary
09:35:57 14272 INFO  Active node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) found failover candidate node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) being promoted (currently "prepare_promotion"/"wait_primary")
09:35:57 14272 INFO  Setting goal state of node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to join_secondary after node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) got selected as the failover candidate.
09:35:57 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ join_secondary
09:35:58 14272 INFO  New state is reported by node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): "join_secondary"
09:35:58 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): join_secondary ➜ join_secondary
09:35:59 14272 INFO  New state is reported by node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): "wait_primary"
09:35:59 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): wait_primary ➜ wait_primary
09:36:00 14272 INFO  Setting goal state of node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to secondary after node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) converged to wait_primary.
09:36:00 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): join_secondary ➜ secondary
09:36:00 14272 INFO  New state is reported by node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): "secondary"
09:36:00 14272 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ secondary
09:36:00 14272 INFO  Setting goal state of node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432) to primary now that at least one secondary candidate node is healthy.
09:36:00 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): wait_primary ➜ primary
09:36:00 14272 INFO  New state is reported by node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): "primary"
09:36:00 14272 INFO  New state for node 1 "node_1" (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): primary ➜ primary
</details></pre>
<pre><details><summary>Вывод node_1</summary>
09:35:57 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ report_lsn
09:35:57 13388 INFO  Fetched current list of 1 other nodes from the monitor to update HBA rules, including 1 changes.
09:35:57 13388 INFO  Ensuring HBA rules for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432)
09:35:57 13388 INFO  Monitor assigned new state "report_lsn"
09:35:57 13388 INFO  FSM transition from "secondary" to "report_lsn": Reporting the last write-ahead log location received
09:35:57 13388 INFO  Restarting standby node to disconnect replication from failed primary node, to prepare failover
09:35:57 13388 INFO  Stopping Postgres at "/var/lib/postgresql/master"
09:35:57 13387 INFO  Stopping pg_autoctl postgres service
09:35:57 13387 INFO  /usr/lib/postgresql/13/bin/pg_ctl --pgdata /var/lib/postgresql/master --wait stop --mode fast
09:35:57 13388 INFO  Creating the standby signal file at "/var/lib/postgresql/master/standby.signal", and replication setup at "/var/lib/postgresql/master/postgresql-auto-failover-standby.conf"
09:35:57 13388 INFO  Contents of "/var/lib/postgresql/master/postgresql-auto-failover-standby.conf" have changed, overwriting
09:35:57 13388 INFO  Restarting Postgres at "/var/lib/postgresql/master"
09:35:57 19471 INFO   /usr/lib/postgresql/13/bin/postgres -D /var/lib/postgresql/master -p 5432 -h *
09:35:57 13387 WARN  PostgreSQL was not running, restarted with pid 19471
09:35:57 13388 INFO  Transition complete: current state is now "report_lsn"
09:35:57 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ report_lsn
09:35:57 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ prepare_promotion
09:35:57 13388 INFO  Monitor assigned new state "prepare_promotion"
09:35:57 13388 INFO  FSM transition from "report_lsn" to "prepare_promotion": Stop traffic to primary, wait for it to finish draining.
09:35:57 13388 INFO  Transition complete: current state is now "prepare_promotion"
09:35:57 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ prepare_promotion
09:35:57 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): prepare_promotion ➜ wait_primary
09:35:57 13388 INFO  Monitor assigned new state "wait_primary"
09:35:57 13388 INFO  FSM transition from "prepare_promotion" to "wait_primary": Promoting a Citus Worker standby after having blocked writes from the coordinator.
09:35:57 13388 INFO  Promoting postgres
09:35:58 13388 INFO  Waiting for postgres to promote
09:35:59 13388 INFO  Cleaning-up Postgres replication settings
09:35:59 13388 INFO  Disabling synchronous replication
09:35:59 13388 INFO  Dropping replication slot "pgautofailover_standby_2"
09:35:59 13388 INFO  Transition complete: current state is now "wait_primary"
09:35:59 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): report_lsn ➜ join_secondary
09:35:59 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): join_secondary ➜ join_secondary
09:35:59 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): wait_primary ➜ wait_primary
09:36:00 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): join_secondary ➜ secondary
09:36:00 13388 INFO  New state for node 3 "node_3" (slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432): secondary ➜ secondary
09:36:00 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): wait_primary ➜ primary
09:36:00 13388 INFO  Monitor assigned new state "primary"
09:36:00 13388 INFO  FSM transition from "wait_primary" to "primary": A healthy secondary appeared
09:36:00 13388 INFO  Setting synchronous_standby_names to '*'
09:36:00 13388 INFO  Waiting until standby node has caught-up to LSN 0/11000350
09:36:00 13388 INFO  Standby reached LSN 0/11000400, thus advanced past LSN 0/11000350
09:36:00 13388 INFO  Transition complete: current state is now "primary"
09:36:00 13388 INFO  New state for this node (node 1, "node_1") (master-t.us-central1-a.c.verdant-coyote-294709.internal:5432): primary ➜ primary
</details></pre>
9. Проверяем состояние кластера на мониторинг хосте:
```
postgres@monitor-t:/home/desmond$ /usr/lib/postgresql/13/bin/pg_autoctl show state --pgdata /var/lib/postgresql/monitor
  Name |  Node |                                                    Host:Port |        LSN | Reachable |       Current State |      Assigned State
-------+-------+--------------------------------------------------------------+------------+-----------+---------------------+--------------------
node_1 |     1 | master-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/11000438 |       yes |             primary |             primary
node_3 |     3 | slave2-t.us-central1-a.c.verdant-coyote-294709.internal:5432 | 0/11000438 |       yes |           secondary |           secondary
```
После удаления мастера на `node_2` новым мастером стал хост `node_1`.