## Выполнение домашнего задания по теме "Разворачиваем и настраиваем БД с большими данными"

### Подготовка инфраструктуры

1. Разворачиваем 2 ВМ с Ubuntu 18.04 для установки СУБД.
2. На 1 ВМ устанавливаем PostgreSQL 13:
```
desmond@postgres-9-1:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
3. На 2 ВМ устанавливаем Greenplump по гайду https://greenplum.org/install-greenplum-oss-on-ubuntu/

### Подготовка больших данных

1. В GCP BigQuery берем общедоступную базу данных. В моем случае я выбрал базу libraries_io, таблицу dependencies
```
Идентификатор таблицы	
bigquery-public-data:libraries_io.dependencies
Размер таблицы	
11,1 ГБ
Количество строк	
105 811 884
Время создания	
22 авг. 2017 г., 23:29:34
Таблица будет удалена	
Никогда
Последнее изменение	
16 нояб. 2020 г., 17:29:50
Место обработки	
US
```
2. Экспортируем в GCS.
3. На виртуальных машинах в папку `tmp` копируем наши данные:
```
desmond@postgres-9-1:/tmp$ gsutil -m cp -R gs://libraries2020 .
```
4. Создадим отдельную базу на обоих ВМ для загрузки данных.
```
postgres@postgres-9-1:/tmp$ createdb libraries
postgres@postgres-9-1:/tmp$ psql libraries
psql (13.1 (Ubuntu 13.1-1.pgdg18.04+1))
Type "help" for help.
```
5. В базе данных создаем точно такую же таблицу:
```
libraries=# create table libraries (
libraries(# id  INTEGER,
libraries(# platform text,
libraries(# project_name text,
libraries(# project_id INTEGER,
libraries(# version_number varchar,
libraries(# version_id INTEGER,
libraries(# dependency_name text,
libraries(# dependency_platform text,
libraries(# dependency_kind text,
libraries(# optional_dependency boolean,
libraries(# dependency_requirements varchar,
libraries(# dependency_project_id INTEGER
libraries(# );
CREATE TABLE
```
6. При помощи `COPY` копируем из `tmp` данные в созданную таблицу:
```
libraries=# COPY libraries (id,
libraries(# platform,
libraries(# project_name,
number,
version_id,
dependency_name,
dependency_platflibraries(# project_id,
libraries(# version_number,
libraries(# version_id,
libraries(# dependency_name,
libraries(# dependency_platform,
libraries(# dependency_kind,
libraries(# optional_dependency,
libraries(# dependency_requirements,
libraries(# dependency_project_id)
libraries-# FROM PROGRAM 'awk FNR-1 /tmp/libraries2020/*.csv | cat' DELIMITER ',' CSV HEADER;
COPY 105811883
```


### Тестирование скорости обработки больших данных

1. Первым делом попробуем протестировать обработку через сам GCP BigQuery:

![BigQuery](https://github.com/apovyshev/PostgreSQL/blob/main/09.BigData/BigQuery.png)

Как видим, запрос обработался всего за 4,4 сек (обработано 3 ГБ).

2. Проверим выполнение запроса в PostgreSQL 13:
```
libraries=# \timing on
libraries=# SELECT project_name, count(platform) as c
libraries-# FROM libraries
libraries-# group by project_name
libraries-# order by c DESC;
Time: 946866.800 ms (15:46.867)
```
Запрос обрабатывался намного дольше при стандартных настройках PostgreSQL.

3. Проверка запроса в Greenplum:
```
libraries=# \timing on
Timing is on.
libraries=# SELECT project_name, count(platform) as c
libraries-# FROM libraries
libraries-# group by project_name
libraries-# order by c DESC;
Time: 372200.060 ms 
```
Время выполнения приблизительно 6.2 минуты.

4. Протестируем 2 СУБД на одной машине, установив обе.
```
# PostgreSQL
libraries=# select platform, version_number, count(dependency_name) as c
libraries-# from libraries
libraries-# group by platform, version_number
libraries-# order by platform
libraries-# ;
Time: 508533.086 ms (08:28.533)
---
# Greenplum
libraries=# select platform, version_number, count(dependency_name) as c
libraries-# from libraries
libraries-# group by platform, version_number
libraries-# order by platform;
Time: 460525.968 ms
```
Если PostgreSQL обрабатывал запрос примерно 8.4 минуты, то Greenplum справился с ним за ~ 7.6.