## Выполнение домашнего задания по теме "Секционирование"

### Подготовка инфраструктуры, базы данных

1. Разворачиваем ВМ с Ubuntu 20.04 для установки СУБД.
2. Устанавливаем PostgreSQL 13:
```
desmond@postgres-11:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
3. Качаем базу demo с https://edu.postgrespro.ru/bookings.pdf - demo_medium.zip
4. Добавляем файл в GCS.
5. На виртуальных машинах в папку `tmp` копируем наши данные:
```
desmond@postgres-10:/tmp$ gsutil -m cp -R gs://flights2021 .
```
6. Копируем бэкап в корневую папку postgres и восстанавливаем базу demo из бэкапа.

### Секционирование по диапазону значений

1. Для секционирования по диапазону значений создадим отдельную таблицу `flights_range`:
```
demo=# create table flights_range (
demo(# flight_id serial,
demo(# flight_no char(6),
demo(# scheduled_departure timestamptz,
demo(# scheduled_arrival timestamptz,
demo(# departure_airport char(3),
demo(# arrival_airport char(3),
demo(# status varchar(20),
demo(# aircraft_code char(3),
demo(# actual_departure timestamptz,
demo(# actual_arrival timestamptz
demo(# ) partition by range (scheduled_departure);
CREATE TABLE
```
2. Создаем секции для дальнейшей загрузки данных. Создадим секции для августа и сентября. Также, не забываем создать дефолтную секцию для всех остальных значений:
```
demo=# create table flights_range_08 partition of flights_range for values from ('2016-08-01 00:00:00+00') to ('2016-08-31 23:59:59+00');
CREATE TABLE
demo=# create table flights_range_09 partition of flights_range for values from ('2016-09-01 00:00:00+00') to ('2016-09-30 23:59:59+00');
CREATE TABLE
demo=# create table flights_range_default partition of flights_range default;
CREATE TABLE
```
3. Копируем все данные из bookings.flights в новую таблицу:
```
demo=# insert into flights_range (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival) select * from bookings.flights;
INSERT 0 65664
```
4. Проверяем, что наши данные были разнесены по секциям корректно:
```
demo=# select count(*) from bookings.flights where scheduled_departure between ('2016-08-01 00:00:00+00') and ('2016-08-31 23:59:59+00');
 count
-------
 16853
(1 row)

demo=# select count(*) from flights_range_08;
 count
-------
 16853
(1 row)

demo=# select count(*) from bookings.flights where scheduled_departure between ('2016-09-01 00:00:00+00') and ('2016-09-30 23:59:59+00');
 count
-------
 16286
(1 row)

demo=# select count(*) from flights_range_09;
 count
-------
 16286
(1 row)
```
Как видим, данные были разнесены по отдельным секциям согласно заданному промежутку.

### Секционирование по списку значений

1. Для секционирования по списку значений создадим отдельную таблицу `flights_list1`:
```
demo=# create table flights_list1 (
demo(# flight_id serial,
demo(# flight_no char(6),
demo(# scheduled_departure timestamptz,
demo(# scheduled_arrival timestamptz,
demo(# departure_airport char(3),
demo(# arrival_airport char(3),
demo(# status varchar(20),
demo(# aircraft_code char(3),
demo(# actual_departure timestamptz,
demo(# actual_arrival timestamptz
demo(# ) partition by list (departure_airport);
CREATE TABLE
```
2. Создаем секции для дальнейшей загрузки данных. Создадим секции для Москвы и Санкт-Петербурга. Также, не забываем создать дефолтную секцию для всех остальных значений:
```
demo=# create table flights_list1_dme partition of flights_list1 for values in ('DME');
CREATE TABLE
demo=# create table flights_list1_led partition of flights_list1 for values in ('LED');
CREATE TABLE
demo=# create table flights_list1_default partition of flights_list1 default;
CREATE TABLE
```
3. Копируем все данные из bookings.flights в новую таблицу:
```
demo=# insert into flights_list1 (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival) select * from bookings.flights;
INSERT 0 65664
```
4. Проверяем, что наши данные были разнесены по секциям корректно:
```
demo=# select count(*) from bookings.flights where departure_airport = 'DME';
 count
-------
  6376
(1 row)

demo=# select count(*) from flights_list1_dme;
 count
-------
  6376
(1 row)

demo=# select count(*) from bookings.flights where departure_airport = 'LED';
 count
-------
  3769
(1 row)

demo=# select count(*) from flights_list1_led;
 count
-------
  3769
(1 row)
```

5. Секцию можно также разбить на дополнительные секции. Создадим еще одну таблицу `flights_list2`:
```
demo=# create table flights_list2 (
demo(# flight_id serial,
demo(# flight_no char(6),
demo(# scheduled_departure timestamptz,
demo(# scheduled_arrival timestamptz,
demo(# departure_airport char(3),
demo(# arrival_airport char(3),
demo(# status varchar(20),
demo(# aircraft_code char(3),
demo(# actual_departure timestamptz,
demo(# actual_arrival timestamptz
demo(# ) partition by list (status);
CREATE TABLE
```
6. Создадим отдельные секции по статусу, а затем создадим еще секции по диапазону значений:
```
demo=# create table flights_list2_delayed_cancelled partition of flights_list2 for values in ('Delayed', 'Cancelled');
CREATE TABLE
demo=# create table flights_list2_scheduled_ontime partition of flights_list2 for values in ('Scheduled', 'On Time');
CREATE TABLE
demo=# create table flights_list2_arrived partition of flights_list2 for values in ('Arrived') partition by range (actual_arrival);
CREATE TABLE
demo=# create table flights_list2_arrived_08 partition of flights_list2_arrived for values from ('2016-08-01 00:00:00+00') to ('2016-08-31 23:59:59+00');
CREATE TABLE
demo=# create table flights_list2_arrived_09 partition of flights_list2_arrived for values from ('2016-09-01 00:00:00+00') to ('2016-09-30 23:59:59+00');
CREATE TABLE
demo=# create table flights_list2_arrived_default partition of flights_list2_arrived default;
CREATE TABLE
demo=# create table flights_list2_default partition of flights_list2 default;
CREATE TABLE
```
7. Добавляем данные:
```
demo=# insert into flights_list2 (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival) select * from bookings.flights;
INSERT 0 65664
```
8. Проверяем соответствие содержимого:
```
demo=# select count(*) from bookings.flights where status in ('Arrived');
 count
-------
 49235
(1 row)

demo=# select count(*) from flights_list2_arrived;
 count
-------
 49235
(1 row)

demo=# select count(*) from bookings.flights where status in ('Arrived') and actual_arrival between ('2016-08-01 00:00:00+00') and ('2016-08-31 23:59:59+00');
 count
-------
 16846
(1 row)

demo=# select count(*) from flights_list2_arrived_08;
 count
-------
 16846
(1 row)
```
Все данные распределились корректно.

### Секционирование по хэшу

1. Для секционирования по списку значений создадим отдельную таблицу `flights_hash`:
```
demo=# create table flights_hash (
demo(# flight_id serial,
demo(# flight_no char(6),
demo(# scheduled_departure timestamptz,
demo(# scheduled_arrival timestamptz,
demo(# departure_airport char(3),
demo(# arrival_airport char(3),
demo(# status varchar(20),
demo(# aircraft_code char(3),
demo(# actual_departure timestamptz,
demo(# actual_arrival timestamptz
demo(# ) partition by hash (flight_id);
CREATE TABLE
```
2. Нашу таблицу будем делить на 5 секций:
```
demo=# create table flights_hash1 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 0);
CREATE TABLE
demo=# create table flights_hash2 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 1);
CREATE TABLE
demo=# create table flights_hash3 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 2);
CREATE TABLE
demo=# create table flights_hash4 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 3);
CREATE TABLE
demo=# create table flights_hash5 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 4);
CREATE TABLE
```
3. Загружаем данные:
```
demo=# insert into flights_hash (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival) select * from bookings.flights;
INSERT 0 65664
```
4. Делаем проверку:
```
demo=# select count(*) from flights_hash1;
 count
-------
 13072
(1 row)

demo=# select count(*) from flights_hash2;
 count
-------
 13336
(1 row)

demo=# select count(*) from flights_hash3;
 count
-------
 13108
(1 row)

demo=# select count(*) from flights_hash4;
 count
-------
 13104
(1 row)

demo=# select count(*) from flights_hash5;
 count
-------
 13044
(1 row)
```
Как видим, данные в таблице были разбиты на примерно равные части.
