## Выполнение домашнего задания по второй теме "SQL и реляционные СУБД. Введение в PostgreSQL"

### Создаем проект в GCP
![NewProject](https://github.com/apovyshev/PostgreSQL/blob/main/01.Transactions/NewProject.PNG)

### Предоставляем права Project Editor для  postgres202010@gmail.com
![TheAccessProvided](https://github.com/apovyshev/PostgreSQL/blob/main/01.Transactions/TheAccessProvided.PNG)

### Создаем экземпляр VM в CGP
![TheInstanceCreated](https://github.com/apovyshev/PostgreSQL/blob/main/01.Transactions/TheInstanceCreated.PNG)

### Подключение к VM через ssh
1. Генерируем пару ключей через команду `ssh-keygen`.
2. Копируем наш публичный ключ, находящийся в файле ~/.ssh/id_rsa.pub и добавляем его в Метаданные нашего экземпляра VM.
3. При помощи команды `ssh user_name@ip_address` удаленно подключаемся к VM, где `user_name` - имя пользователя, под которым происходит подключение с локального хоста, а `ip_address` - IP нашей виртуально машины.

### Установка PostgreSQL 13
1. Для установки PostgreSQL последней версии необходимо выполнить следующую команду:
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
2. После установки убеждаемся, что кластер запустился пр помощи `pg_lsclusters` . Если кластер запущен, получаем примерно следующий вывод:
```
desmond@postgresql-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```

### Создание новой базы данных и выполнение транзакций
1. В первой сессии под пользователем `postgres` подключаемся к базе данных при помощи команды `sudo -i -u postgres psql`.
2. Создаем новую базу "transactions" при помощи команды `create database transactions;` и подключаемся к ней, выполнив `\c transactions`.
3. Запускаем вторую сессию и также подключаемся к базе "transactions" при помощи `sudo -i -u postgres psql transactions`.
4. В обоих сессиях отключаем "auto commit" командой `\set AUTOCOMMIT OFF`. Проверить статус можно командой `\echo :AUTOCOMMIT`.
5. В первой сессии создаем новую таблицу "persons" и наполняем ее данными:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
6. При помощи команды `select * from persons;` проверяем содержимое таблицы в первой и второй сессии:
```
transactions=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Как видим, информация появилась в обоих сессиях.

7. Проверяем текущий уровень изоляции при помощи `show transaction isolation level;` :
```
transactions=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

8. Не меняя уровень изоляции (оставляем дефолтный "read committed"), в первой сессии добавляем новую запись без выполнения `commit;` и проверяем содержимое таблицы:
```
transactions=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
transactions=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

9. Далее делаем проверку `select * from persons;` во второй сессии и видим, что данные не подтянулись:
```
transactions=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Это произошло потому, что в первой сессии мы не завершили транзакцию, выполнив `commit;`. После завершения транзакции в первой сессии, выполнив команду `commit;`, во второй сессии мы получаем необходимые данные:
```
transactions=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Данные подтянулись из-за того, что в первой сессии транзакция была завершена командой `commit;`. 
Это особенность уровня изоляции "read committed": пока не будет завершена транзакция сессии, данные не будут обновлены.  
В транзакции второй сесси запрос SELECT видит только те данные, которые были зафиксированы до начала запроса; он никогда не увидит незафиксированных данных или изменений, внесённых в процессе выполнения запроса параллельными транзакциями (в нашем случае в первой сессии). Прооизошло так называемое "фантомное чтение".

10. Изменяем уровень изоляции в обоих сессиях на "repeatable read", выполнив команду `set transaction isolation level repeatable read;` :
```
transactions=*# set transaction isolation level repeatable read;
SET
transactions=*# show transaction isolation level;
 transaction_isolation
-----------------------
 repeatable read
(1 row)
```

11. В первой сессии добавляем новую запись в таблицу без выполнения `commit;`:
```
transactions=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
transactions=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

12. Проверяем данные таблицы во второй сессии:
```
transactions=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Как и в случае с уровнем изоляции "read committed", наши данные не подтянулись во второй сессии до завершения транзакции в первой. 
Однако, завершив транзакцию в первой сессии, при проверке данных во второй мы получаем тот же результат. 
В режиме Repeatable Read видны только те данные, которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, произведённые другими транзакциями в процессе выполнения данной транзакции. 
Завершив транзакцию во второй сессии мы получим обновленную информацию, тем самым изменяя уровень изоляции на дефолтный:
```
transactions=*# commit;
COMMIT
transactions=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

