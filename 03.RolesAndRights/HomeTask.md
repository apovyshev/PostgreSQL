## Выполнение домашнего задания по второй теме "Логический уровень PostgreSQL"

### Создаем экземпляр VM в CGP
![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/03.RolesAndRights/VMCreated.PNG)

### Устанавливаем PostgreSQL 13
```
desmond@postgres-3:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```

### Операции с таблтицами, схемами, пользователями и правами

1. Подключаемся к нашему кластеру под пользователем `postgres`:
```
desmond@postgres-3:~$ sudo -i -u postgres psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```
2. Создаем новую базу `testdb`:
```
postgres=# create database testdb;
CREATE DATABASE
```
3. Подключаемся к новой базе под пользователем `postgres` и создаем новую схему `testnm`:
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
```
4. Cоздаем новую таблицу t1 с одной колонкой c1 типа integer и вставляем в нее строку со значением c1=1:
```
testdb=# create table t1(c1 int);
CREATE TABLE
testdb=# insert into t1(c1) values (1);
INSERT 0 1
testdb=# select * from t1;
 c1
----
  1
(1 row)
```
5. Создаем новую роль `readonly`:
```
testdb=# create role readonly;
CREATE ROLE
```
6. Предоставляем новой роли `readonly` права на подключение к базе данных `testdb`:
```
testdb=# grant connect on database testdb to readonly;
GRANT
```
7. Предоставляем новой роли `readonly` права на использование схемы `testnm`:
```
testdb=# grant usage on schema testnm to readonly;
GRANT
```
8. Предоставляем новой роли `readonly` права на `select` для всех таблиц схемы `testnm`:
```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```
9. Создадим пользователя `testread` с паролем `test123`:
```
testdb=# create role testread with password 'test123';
CREATE ROLE
```
10. Даем роль `readonly` пользователю `testread`:
```
testdb=# grant readonly to testread;
GRANT ROLE
```
11. Подключаемся к базе данных `testdb` под пользователем `testread`:
```
testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>
```

NOTE!!!

Для того, чтобы пользователь `testread` мог подключаться к базе данных, необходимо произвести следующие настройки:

- изменить тип подключения для пользователей с `peer` на `md5`:
```
desmond@postgres-3:~$ sudo vim /etc/postgresql/13/main/pg_hba.conf
---
# "local" is for Unix domain socket connections only
local   all             all                                     md5
---
desmond@postgres-3:~$ sudo pg_ctlcluster 13 main reload
```
- предоставить пользователю права на `login`:
```
testdb=# alter role testread with login;
ALTER ROLE
```

12. Пробуем посмотреть содержимое нашей таблицы `t1`:
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
Почему же так вышло? 
Все дело в том, что нашу таблицу `t1` мы создавали в схеме `public`.
```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
 ````
 Согласно `search_path`, таблицы сначала создаются в схеме `"$user"`, соответствующей имени пользователя, под которым мы работаем в базе данных, и, если одноименной схемы нет, то таблицы попадают в схему `public`.
Чтобы мы получили доступ к таблице `t1` под пользователем `testread` нам необходимо было либо создавать нашу таблицу в схеме `testnm`, либо предоставлять дополнительные права на таблицы схемы `public`.

13. Подключаемся к нашей базе `testdb` под пользователей `postgres` и удаляем нашу таблицу `t1`:
```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# drop table t1;
DROP TABLE
testdb=# \dt
Did not find any relations.
```
14. Создадим заново таблицу `t1`, но уже в схеме `testnm`:
```
testdb=# create table testnm.t1(c1 int);
CREATE TABLE
testdb=# insert into testnm.t1(c1) values (1);
INSERT 0 1
```
15. Подключаемся под нашим пользователем `testread` и проверяем содержимое таблицы `t1`:
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Почему же так вышло? Ведь права мы выдавали на шаге 8 при помощи команды `grant select on all tables in schema testnm to readonly;`. Все дело в том, что в тот момент мы выдавали права на те таблицы, которые были созданы на момент выдачи прав.
Чтобы избежать постоянных операций с выдачей прав после каждого создания новой таблицы, необходимо определить права доступа по умолчанию (более подробно смотри на странице https://postgrespro.ru/docs/postgrespro/12/sql-alterdefaultprivileges):
```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```
16. Подключаемся к пользователю `testread` и проверяем наши данные:
```
testdb=# \c - testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Снова ошибка! Как так-то?
Выполнив команду `alter default privileges in schema testnm grant select on tables to readonly;` мы создали правило для новых таблиц базы данных, но для того, чтобы мы смогли видеть существующие таблицы, необходимо еще раз выполнить `grant select on all tables in schema testnm to readonly;`:

```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
testdb=# \c - testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
17. Пробуем создать новую таблицу под пользователем `testread` и добавить данные в нее:
```
testdb=> create table t2(c1 int);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
```
Почему так получилось? Ведь мы не выдавали прав на создание таблиц и прав на `insert` для этого пользователя. 
Опять же, ответ на вопрос кроется в `search_path`. Если мы не указываем конкретную схему при создании таблицы, то первым делом, таблица будет создаваться в схеме `"$user"` и, если такой нет, то создаст ее в `public`. При этом, доступ к схеме `public` предоставляемся всем пользователям по умолчанию и, если есть необходимые права на подключение к базе, любой пользователь может создавать там объекты.
```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | testread
(1 row)
```
Но еси мы попробуем создать таблицу в схеме `testnm`, то увидим, что прав на это у нас нет:

```
testdb=> create table testnm.t2(c1 int);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 int);
                     ^
```
18. Уберем право у роли `public` на `create` в схеме `public`, а также все права в базе данных в базе данных `testdb` для пользователя `public`:
```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# revoke create on schema public from public;
REVOKE
testdb=# revoke all on database testdb from public;
REVOKE
```
19. Попробуем создать новую таблицу под пользователем `testread`:
```
testdb=# \c - testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
```
Данный результат мы получили в следствие того, что отключили создание объектов в схеме `public`. 

20. Наполняем данными нашу таблицу `t2`:
```
testdb=> insert into t2 values (2);
INSERT 0 1
testdb=> select * from t2;
 c1
----
  2
  2
(2 rows)
```
Наполнить таблицу у нас получилось, так как мы отняли только право на создание новых таблиц в схеме `public`.