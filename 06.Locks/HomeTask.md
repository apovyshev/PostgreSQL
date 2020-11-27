## Выполнение домашнего задания по теме "Механизм блокировок"

### Настройка журналирования сообщений о блокировках
Настроить журналирование сообщений о блокировках, удерживаемых 200ms можно двумя способами:изменить параметр `log_min_duration_statement` и тогда в журнал будут идти любые операции, которые выполнялись более заданного времени, либо включить `log_lock_waits` и указать `deadlock_timeout`. Разберем оба варианта, начиная с `log_min_duration_statement`.

### 1. log_min_duration_statement
1. Создаем отдельную базу `locks` для воспроизведения блокировок и сразу подключимся к ней:
```
postgres=# create database locks;
CREATE DATABASE
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
```
2. Создадим таблицу и наполним данными:
```
locks=# CREATE TABLE accounts(id integer, amount numeric);
CREATE TABLE
locks=# INSERT INTO accounts VALUES (1,2000.00), (2,2000.00), (3,2000.00);
INSERT 0 3
locks=# select * from accounts;
 id | amount
----+---------
  1 | 2000.00
  2 | 2000.00
  3 | 2000.00
(3 rows)
```
3. Меняем параметр `log_min_duration_statement`:
```
postgres=# alter system set log_min_duration_statement = 200;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
postgres=# show log_min_duration_statement;
 log_min_duration_statement
----------------------------
 200ms
(1 row)
```
4. Воспроизведем блокировку:
- В первом окне выполняем операцию с командой Update на определенное поле:
```
### this transaction should be executed in the primary window 
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;
UPDATE 1
```
- Во втором окне выполним ту же операцию Update с тем же полем:
```
### this transaction should be executed in the secondary window 
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
|
```

Во втором окне транзакция ожидает пока завершится транзакция из первого

- Завершим транзакцию в первом:
```
### this transaction should be executed in the primary window
locks=*# commit;
COMMIT
```

- Во втором окне блокировка пропала и транзакция может быть также завершена:
```
### this transaction should be executed in the secondary window
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
UPDATE 1
locks=*# commit;
COMMIT
```
5. Проверим лог из журнала на предмет появления в нем записи блокировки:
```
desmond@postgres-6:~$ tail -n 3 /var/log/postgresql/postgresql-13-main.log
2020-11-27 09:22:07.599 UTC [12813] LOG:  received SIGHUP, reloading configuration files
2020-11-27 09:22:07.599 UTC [12813] LOG:  parameter "log_min_duration_statement" changed to "200"
2020-11-27 09:28:01.617 UTC [14393] postgres@locks LOG:  duration: 16315.155 ms  statement: UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
```
Как видим, в журнал попала наша транзакция, которая ожидала блокировку.


### 2. log_lock_waits и deadlock_timeout

1. Меняем параметры для `log_lock_waits` и `deadlock_timeout`:
```
 alter system set log_min_duration_statement = -1;
ALTER SYSTEM
locks=# alter system set log_lock_waits = on;
ALTER SYSTEM
locks=# alter system set deadlock_timeout = 200;
ALTER SYSTEM
locks=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

locks=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

locks=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```
2. Воспроизведем блокировку:
- В первом окне выполняем операцию с командой Update на определенное поле:
```
### this transaction should be executed in the primary window 
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;
UPDATE 1
```
- Во втором окне выполним ту же операцию Update с тем же полем:
```
### this transaction should be executed in the secondary window 
locks=# BEGIN;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
|
```

Во втором окне транзакция ожидает пока завершится транзакция из первого

- Завершим транзакцию в первом:
```
### this transaction should be executed in the primary window
locks=*# commit;
COMMIT
```

- Во втором окне блокировка пропала и транзакция может быть также завершена:
```
### this transaction should be executed in the secondary window
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
UPDATE 1
locks=*# commit;
COMMIT
```
3. Проверим лог из журнала на предмет появления в нем записи блокировки:
```
tail -n 7 /var/log/postgresql/postgresql-13-main.log
2020-11-27 09:34:21.052 UTC [14412] postgres@locks LOG:  process 14412 still waiting for ShareLock on transaction 490 after 200.157 ms
2020-11-27 09:34:21.052 UTC [14412] postgres@locks DETAIL:  Process holding the lock: 14393. Wait queue: 14412.
2020-11-27 09:34:21.052 UTC [14412] postgres@locks CONTEXT:  while updating tuple (0,5) in relation "accounts"
2020-11-27 09:34:21.052 UTC [14412] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
2020-11-27 09:34:39.575 UTC [14412] postgres@locks LOG:  process 14412 acquired ShareLock on transaction 490 after 18722.935 ms
2020-11-27 09:34:39.575 UTC [14412] postgres@locks CONTEXT:  while updating tuple (0,5) in relation "accounts"
2020-11-27 09:34:39.575 UTC [14412] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
```
Блокировка отобразилась и в этом случае, но уже с большими деталями о времени ожидания, а также процессами блокировки.

### Блокировки в представлении pg_locks

1. Перед тем, как начать тестирование блокировок, узнаем pid для каждой сессии, выполнив команду `SELECT pg_backend_pid();` в каждом из 3х окон:
```
locks=# SELECT pg_backend_pid();
# the primary window
pg_backend_pid
----------------
          14393
(1 row)
# the secondary window
pg_backend_pid
----------------
          14412
(1 row)
# the third window
 pg_backend_pid
----------------
          15867
(1 row)
```
2. Посмотрим, какие данные хранятся в pg_locks до начала теста (первое окно):
```
locks=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 14393;
  locktype  | relation | virtxid | xid |      mode       | granted
------------+----------+---------+-----+-----------------+---------
 relation   | pg_locks |         |     | AccessShareLock | t
 virtualxid |          | 5/27    |     | ExclusiveLock   | t
(2 rows)
```
В нашем случае мы видим следующие блокировки:
- на команду `pg_locks` в режиме `AccessShareLock` - вызвана тем, что мы в данный момент просматриваем статистику через `selectёж
- блокировка типа `virtualxid`, так как транзакция всегда удерживает исключительную (ExclusiveLock) блокировку собственного номера, а данном случае — виртуального.

3. Начнем одну и ту же транзакцию в каждом окне:
```
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 1;
```
Как и ожидалось, в первом окне транзакция выполнится `UPDATE 1`, а в двух других сессиях будут заблокированы.

4. Проверим информацию о блокировках по всем 3 pid:
```
locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 14393;
   locktype    | relation | virtxid | xid |       mode       | granted
---------------+----------+---------+-----+------------------+---------
 relation      | pg_locks |         |     | AccessShareLock  | t
 relation      | accounts |         |     | RowExclusiveLock | t
 virtualxid    |          | 5/28    |     | ExclusiveLock    | t
 transactionid |          |         | 495 | ExclusiveLock    | t
(4 rows)
```
Помимо `relation | pg_locks` и `virtualxid` у нас появили 2 дополнительные записи:
- `|relation      | accounts |         |     | RowExclusiveLock | t|` - означает, что в данный момент мы выполнили блокировку на таблицу `accounts` в режиме `RowExclusiveLock`, который как раз соответствует нашей операции `UPDATE`. Параметр `t` означает, что блокировка успешно прошла.
- `|transactionid |          |         | 492 | ExclusiveLock    | t|` - исключительная блокировка транзакции 492, которая появилась, как только транзакция начала изменять данные.

Проверим блокировки pid второго окна:
```
locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 14412;
   locktype    | relation | virtxid | xid |       mode       | granted
---------------+----------+---------+-----+------------------+---------
 relation      | accounts |         |     | RowExclusiveLock | t
 virtualxid    |          | 3/40    |     | ExclusiveLock    | t
 transactionid |          |         | 496 | ExclusiveLock    | t
 transactionid |          |         | 495 | ShareLock        | f
 tuple         | accounts |         |     | ExclusiveLock    | t
(5 rows)
```
Мы видим, что по сравнению с информацией о блокировках в рамках собственного pid, для pid из второго окна появились дополнительные типы:
- `|transactionid |          |         | 496 | ExclusiveLock    | t|` - запрос на блокировку для выполнения транзакции `UPDATE`;
- `|transactionid |          |         | 495 | ShareLock        | f|` - указывает на то, что блокировка на транзакцию не может быть выполнена, так как в данный момент это поле уже заблокировано другой транзакцией с xid 495. Это транзакция из перовго окна, которая выполняет `UPDATE`;
- `|tuple         | accounts |         |     | ExclusiveLock    | t|` - говорит о том, что был создан tuple для ожидания завершения транзакции, которая сейчас блокирует строку. Это говорит о том, что данная транзакция стоит первой в очереди.


И теперь посмотрим блокировки нашей третьей сесси:
```
locks=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 15867;
   locktype    | relation | virtxid | xid |       mode       | granted
---------------+----------+---------+-----+------------------+---------
 relation      | accounts |         |     | RowExclusiveLock | t
 virtualxid    |          | 6/4     |     | ExclusiveLock    | t
 transactionid |          |         | 497 | ExclusiveLock    | t
 tuple         | accounts |         |     | ExclusiveLock    | f
(4 rows)
```
Новых типов блокировок нет, только изменение уже описанных ранее:
- `|relation      | accounts |         |     | RowExclusiveLock | t|` - блокировка на таблицу accounts;
- `|transactionid |          |         | 497 | ExclusiveLock    | t|` - новая транзакция на `UPDATE`;
- `|tuple         | accounts |         |     | ExclusiveLock    | f|` - в данном случае не было изменения `tuple`, так как  `tuple` была ранее создана в сессии 2. Поэтому, наш запрос просто стал в очередь на исполнение.

Проверим очередь на выполнение транзакций:
```
locks=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
locks-*# FROM pg_locks WHERE relation = 'accounts'::regclass;
 locktype |       mode       | granted |  pid  | wait_for
----------+------------------+---------+-------+----------
 relation | RowExclusiveLock | t       | 15867 | {14412}
 relation | RowExclusiveLock | t       | 14393 | {}
 relation | RowExclusiveLock | t       | 14412 | {14393}
 tuple    | ExclusiveLock    | f       | 15867 | {14412}
 tuple    | ExclusiveLock    | t       | 14412 | {14393}
(5 rows)
```
Из этого всего видно, что для выполнения операции с pid 14412 (вторая сессия) необходимо дождаться выполнения pid 14393 (сессия 1), а для pid 15867 (третья сессия) необходимо дождаться выполнения 14412 (вторая).

### Взаимоблокировка
1. Для воспроизведения взаимоблокировки между тремя транзакциями используем перевод денег с акканунта с id = 1 на аккаунт id = 2, с id = 2 на id = 3, с id = 3 на id = 1. Для начала снимаем со всех счетов сумму, которую необходимо перевести на другой аккаунт:
```
locks=# truncate accounts;
locks=# select * from accounts;
 id | amount
----+---------
  1 | 2000.00
  2 | 2000.00
  3 | 2000.00
(3 rows)
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;
UPDATE 1
---
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 50.00 WHERE id = 2;
UPDATE 1
---
locks=# begin;
BEGIN
locks=*# UPDATE accounts SET amount = amount - 200.00 WHERE id = 3;
UPDATE 1
```

2. При выполнении транзакции блокировки произошли на все счета. Если начнем выполнять пополнение счетов получателей, то транзакции будут ожидать завершения блокировок:
```
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;

---
locks=*# UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;

---
```
И после того, как мы выполним перевод средств с id = 3 на id = 1, произойтет `deadlock`:
```
locks=*# UPDATE accounts SET amount = amount + 200.00 WHERE id = 1;
ERROR:  deadlock detected
DETAIL:  Process 18275 waits for ShareLock on transaction 515; blocked by process 14393.
Process 14393 waits for ShareLock on transaction 516; blocked by process 14412.
Process 14412 waits for ShareLock on transaction 517; blocked by process 18275.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```
После этого перевод с id = 2 на id = 3 выполнится, а с id = 1 на id = 2 все еще будет в статусе ожидания:
```
locks=*# UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;
UPDATE 1
---
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;

---
```
После коммита транзакции `UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;` перевод между id = 1 и id = 2 выполнится:
```
locks=*# UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;
UPDATE 1
```
Но после коммита перевода с id = 3 на id = 1 произойдет `rollback`:
```
locks=!# commit;
ROLLBACK
locks=# select * from accounts;
 id | amount
----+---------
  1 | 1900.00
  3 | 2050.00
  2 | 2050.00
(3 rows)
```
3. Проверим журнал, попал ли наш `deadlock` в запись:
```
desmond@postgres-6:~$ tail -n 30 /var/log/postgresql/postgresql-13-main.log
2020-11-27 14:16:00.713 UTC [14412] postgres@locks STATEMENT:  select * from accounts;
2020-11-27 14:20:13.272 UTC [14393] postgres@locks LOG:  process 14393 still waiting for ShareLock on transaction 516 after 200.152 ms
2020-11-27 14:20:13.272 UTC [14393] postgres@locks DETAIL:  Process holding the lock: 14412. Wait queue: 14393.
2020-11-27 14:20:13.272 UTC [14393] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2020-11-27 14:20:13.272 UTC [14393] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;
2020-11-27 14:20:29.640 UTC [14412] postgres@locks LOG:  process 14412 still waiting for ShareLock on transaction 517 after 200.164 ms
2020-11-27 14:20:29.640 UTC [14412] postgres@locks DETAIL:  Process holding the lock: 18275. Wait queue: 14412.
2020-11-27 14:20:29.640 UTC [14412] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2020-11-27 14:20:29.640 UTC [14412] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;
2020-11-27 14:20:43.703 UTC [18275] postgres@locks LOG:  process 18275 detected deadlock while waiting for ShareLock on transaction 515 after 200.146 ms
2020-11-27 14:20:43.703 UTC [18275] postgres@locks DETAIL:  Process holding the lock: 14393. Wait queue: .
2020-11-27 14:20:43.703 UTC [18275] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2020-11-27 14:20:43.703 UTC [18275] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE id = 1;
2020-11-27 14:20:43.703 UTC [18275] postgres@locks ERROR:  deadlock detected
2020-11-27 14:20:43.703 UTC [18275] postgres@locks DETAIL:  Process 18275 waits for ShareLock on transaction 515; blocked by process 14393.
        Process 14393 waits for ShareLock on transaction 516; blocked by process 14412.
        Process 14412 waits for ShareLock on transaction 517; blocked by process 18275.
        Process 18275: UPDATE accounts SET amount = amount + 200.00 WHERE id = 1;
        Process 14393: UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;
        Process 14412: UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;
2020-11-27 14:20:43.703 UTC [18275] postgres@locks HINT:  See server log for query details.
2020-11-27 14:20:43.703 UTC [18275] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2020-11-27 14:20:43.703 UTC [18275] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 200.00 WHERE id = 1;
2020-11-27 14:20:43.703 UTC [14412] postgres@locks LOG:  process 14412 acquired ShareLock on transaction 517 after 14263.201 ms
2020-11-27 14:20:43.703 UTC [14412] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"
2020-11-27 14:20:43.703 UTC [14412] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 50.00 WHERE id = 3;
2020-11-27 14:31:22.116 UTC [14393] postgres@locks LOG:  process 14393 acquired ShareLock on transaction 516 after 669043.758 ms
2020-11-27 14:31:22.116 UTC [14393] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2020-11-27 14:31:22.116 UTC [14393] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE id = 2;
```
Первое упоминание о `deadlock` уже есть в строчке `2020-11-27 14:20:43.703 UTC [18275] postgres@locks LOG:  process 18275 detected deadlock while waiting for ShareLock on transaction 515 after 200.146 ms`
А далее в детялях видно, что все 3 процесса ожидали выполнения своего соседа:
```
Process 18275 waits for ShareLock on transaction 515; blocked by process 14393.
Process 14393 waits for ShareLock on transaction 516; blocked by process 14412.
Process 14412 waits for ShareLock on transaction 517; blocked by process 18275.
```

### Воспроизведение deadlock одной и той же операцией UPDATE (без WHERE)

1. Для воспроизведения блокировки транзакций `UPDATE` будем использовать курсор. Начнем транзакцию:
```
locks=# begin;
BEGIN
---
locks=# begin;
BEGIN
```
2. Создадим в одном окне курсор, который будет считывать информацию сверху вниз, а второй снизу вверх:
```
locks=*# declare update1 cursor for select * from accounts order by id for update;
DECLARE CURSOR
---
locks=*# declare update2 cursor for select * from accounts order by id desc for update;
DECLARE CURSOR
```
3. Начнем считывать информацию:
```
locks=*# fetch update1;
 id | amount
----+---------
  1 | 1900.00
(1 row)
---
locks=*# fetch update2;
 id | amount
----+---------
  3 | 2050.00
(1 row)
```
```
locks=*# fetch update1;
 id | amount
----+---------
  2 | 2050.00
(1 row)
---
locks=*# fetch update2;

```
Как видим, во втором окне сработала блокировка, так как в первом окне id = 2 уже занят.
```
locks=*# fetch update1;
ERROR:  deadlock detected
DETAIL:  Process 14393 waits for ShareLock on transaction 521; blocked by process 14412.
Process 14412 waits for ShareLock on transaction 520; blocked by process 14393.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,7) in relation "accounts"
locks=!# commit;
ROLLBACK
---
locks=*# fetch update2;
 id | amount
----+---------
  2 | 2050.00
(1 row)

locks=*# commit;
COMMIT
```
После повторного выполнения `fetch update1` сработал `deadlock`.