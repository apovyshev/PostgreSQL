## Выполнение домашнего задания по теме "Работа с индексами, join'ами, статистикой"

### Вариант 2

### Подготовка базы данных и таблиц

Прежде, чем приступить к операциям с соединениями таблиц, создадим новую базу и 3 произвольные таблицы для нее: первая - название футбольного клуба, вторая - страна, к которой относится футбольный клуб, третья - год, в котором побеждала команда в рамках своего внутреннего чемпионата:
```
desmond@postgres-10:~$ sudo su postgres
postgres@postgres-10:/home/desmond$ createdb joins
postgres@postgres-10:/home/desmond$ psql joins
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.
```
Таблица №1
```
create table clubs(id int, name text, country_id int, champ_id int);
insert into clubs(id, name, country_id, champ_id) values (1, 'Chelsea', 1, 6);
insert into clubs(id, name, country_id, champ_id) values (1, 'Chelsea', 1, 5);
insert into clubs(id, name, country_id, champ_id) values (1, 'Chelsea', 1, 10);
insert into clubs(id, name, country_id, champ_id) values (1, 'Chelsea', 1, 15);
insert into clubs(id, name, country_id, champ_id) values (1, 'Chelsea', 1, 17);
insert into clubs(id, name, country_id, champ_id) values (2, 'Arsenal', 1, 2);
insert into clubs(id, name, country_id, champ_id) values (2, 'Arsenal', 1, 4);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 13);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 11);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 9);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 8);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 7);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 3);
insert into clubs(id, name, country_id, champ_id) values (3, 'Manchecter United', 1, 1);
insert into clubs(id, name, country_id, champ_id) values (4, 'Manchester City', 1, 12);
insert into clubs(id, name, country_id, champ_id) values (4, 'Manchester City', 1, 14);
insert into clubs(id, name, country_id, champ_id) values (4, 'Manchester City', 1, 18);
insert into clubs(id, name, country_id, champ_id) values (4, 'Manchester City', 1, 19);
insert into clubs(id, name, country_id, champ_id) values (5, 'Liverpool', 1, 20);
insert into clubs(id, name, country_id) values (6, 'Everton', 1);
insert into clubs(id, name, country_id) values (7, 'Tottenham', 1);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 20);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 17);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 12);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 8);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 7);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 3);
insert into clubs(id, name, country_id, champ_id) values (8, 'Real Madrid', 2, 1);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 19);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 18);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 16);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 15);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 13);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 11);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 10);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 9);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 6);
insert into clubs(id, name, country_id, champ_id) values (9, 'Barcelona', 2, 5);
insert into clubs(id, name, country_id) values (10, 'Sevilla', 2);
insert into clubs(id, name, country_id, champ_id) values (11, 'Valencia', 2, 2);
insert into clubs(id, name, country_id, champ_id) values (11, 'Valencia', 2, 4);
insert into clubs(id, name, country_id, champ_id) values (12, 'Atlético Madrid', 2, 14);
insert into clubs(id, name, country_id) values (13, 'Deportivo La Coruña', 2);
insert into clubs(id, name, country_id) values (14, 'Real Betis', 2);
insert into clubs(id, name, country_id, champ_id) values (15, 'Milan', 3, 11);
insert into clubs(id, name, country_id, champ_id) values (15, 'Milan', 3, 4);
insert into clubs(id, name, country_id, champ_id) values (16, 'Internazionale', 3, 10);
insert into clubs(id, name, country_id, champ_id) values (16, 'Internazionale', 3, 9);
insert into clubs(id, name, country_id, champ_id) values (16, 'Internazionale', 3, 8);
insert into clubs(id, name, country_id, champ_id) values (16, 'Internazionale', 3, 7);
insert into clubs(id, name, country_id, champ_id) values (16, 'Internazionale', 3, 6);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 20);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 19);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 18);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 17);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 16);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 15);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 14);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 13);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 12);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 5);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 3);
insert into clubs(id, name, country_id, champ_id) values (17, 'Juventus', 3, 2);
insert into clubs(id, name, country_id) values (18, 'Fiorentina', 3);
insert into clubs(id, name, country_id) values (19, 'Napoli', 3);
insert into clubs(id, name, country_id, champ_id) values (20, 'Roma', 3, 1);
insert into clubs(id, name, country_id) values (21, 'Parma', 3);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 20);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 19);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 18);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 17);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 16);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 15);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 14);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 13);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 10);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 8);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 6);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 5);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 3);
insert into clubs(id, name, country_id, champ_id) values (22, 'Bayern Munich', 4, 1);
insert into clubs(id, name, country_id, champ_id) values (23, 'Borussia Dortmund', 4, 2);
insert into clubs(id, name, country_id, champ_id) values (23, 'Borussia Dortmund', 4, 11);
insert into clubs(id, name, country_id, champ_id) values (23, 'Borussia Dortmund', 4, 22);
insert into clubs(id, name, country_id) values (24, 'Bayer Leverkusen', 4);
insert into clubs(id, name, country_id) values (25, 'Schalke 04', 4);
insert into clubs(id, name, country_id, champ_id) values (26, 'VfL Wolfsburg', 4, 9);
insert into clubs(id, name, country_id) values (27, 'RB Leipzig', 4);
insert into clubs(id, name, country_id, champ_id) values (28, 'CVfB Stuttgart', 4, 7);
```
Таблица №2
```
create table country (id int, country_name text);
insert into country(id, country_name) values (1, 'England');
insert into country(id, country_name) values (2, 'Spain');
insert into country(id, country_name) values (3, 'Italy');
insert into country(id, country_name) values (4, 'Germany');
```
Таблица №3
```
create table year (id int, year text);
insert into year(id, year) values (1, '2000/2001');
insert into year(id, year) values (2, '2001/2002');
insert into year(id, year) values (3, '2002/2003');
insert into year(id, year) values (4, '2003/2004');
insert into year(id, year) values (5, '2004/2005');
insert into year(id, year) values (6, '2005/2006');
insert into year(id, year) values (7, '2006/2007');
insert into year(id, year) values (8, '2007/2008');
insert into year(id, year) values (9, '2008/2009');
insert into year(id, year) values (10, '2009/2010');
insert into year(id, year) values (11, '2010/2011');
insert into year(id, year) values (12, '2011/2012');
insert into year(id, year) values (13, '2012/2013');
insert into year(id, year) values (14, '2013/2014');
insert into year(id, year) values (15, '2014/2015');
insert into year(id, year) values (16, '2015/2016');
insert into year(id, year) values (17, '2016/2017');
insert into year(id, year) values (18, '2017/2018');
insert into year(id, year) values (19, '2018/2019');
insert into year(id, year) values (20, '2019/2020');
insert into year(id, year) values (21, '2020/2021');
```

### Прямое соединение (Inner Join)

С нашими таблицами реализуем соединение через `inner join`:
```
joins=# select distinct
joins-# clubs.name,
joins-# country.country_name
joins-# from
joins-# clubs
joins-# join country on country.id = clubs.country_id
joins-# order by
joins-# country.country_name,
joins-# clubs.name LIMIT 10 \gx
-[ RECORD 1 ]+------------------
name         | Arsenal
country_name | England
-[ RECORD 2 ]+------------------
name         | Chelsea
country_name | England
-[ RECORD 3 ]+------------------
name         | Everton
country_name | England
-[ RECORD 4 ]+------------------
name         | Liverpool
country_name | England
-[ RECORD 5 ]+------------------
name         | Manchecter United
country_name | England
-[ RECORD 6 ]+------------------
name         | Manchester City
country_name | England
-[ RECORD 7 ]+------------------
name         | Tottenham
country_name | England
-[ RECORD 8 ]+------------------
name         | Bayer Leverkusen
country_name | Germany
-[ RECORD 9 ]+------------------
name         | Bayern Munich
country_name | Germany
-[ RECORD 10 ]------------------
name         | Borussia Dortmund
country_name | Germany
```
Прямое соединение необходимо использовать тогде, когда и с одной таблице, и во сторой есть связующие данные, как в нашем случае есть все данные по клубам и их стране. Если бы мы соединили еще и третью таблицу, то те клубы, которые не были чемпионами за последние 20 сезонов, не попали бы в выборку:

```
joins=# select
joins-# clubs.name,
joins-# country.country_name,
joins-# array_to_string(array_agg(DISTINCT year.year), ', ')
joins-# from
joins-# clubs
joins-# join country on country.id = clubs.country_id
joins-# join year on year.id = clubs.champ_id
joins-# group by
joins-# clubs.name,
joins-# country.country_name
joins-# order by
joins-# country.country_name,
joins-# clubs.name LIMIT 10 \gx
-[ RECORD 1 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Arsenal
country_name    | England
array_to_string | 2001/2002, 2003/2004
-[ RECORD 2 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Chelsea
country_name    | England
array_to_string | 2004/2005, 2005/2006, 2009/2010, 2014/2015, 2016/2017
-[ RECORD 3 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Liverpool
country_name    | England
array_to_string | 2019/2020
-[ RECORD 4 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Manchecter United
country_name    | England
array_to_string | 2000/2001, 2002/2003, 2006/2007, 2007/2008, 2008/2009, 2010/2011, 2012/2013
-[ RECORD 5 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Manchester City
country_name    | England
array_to_string | 2011/2012, 2013/2014, 2017/2018, 2018/2019
-[ RECORD 6 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Bayern Munich
country_name    | Germany
array_to_string | 2000/2001, 2002/2003, 2004/2005, 2005/2006, 2007/2008, 2009/2010, 2012/2013, 2013/2014, 2014/2015, 2015/2016, 2016/2017, 2017/2018, 2018/2019, 2019/2020
-[ RECORD 7 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Borussia Dortmund
country_name    | Germany
array_to_string | 2001/2002, 2010/2011
-[ RECORD 8 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | CVfB Stuttgart
country_name    | Germany
array_to_string | 2006/2007
-[ RECORD 9 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | VfL Wolfsburg
country_name    | Germany
array_to_string | 2008/2009
-[ RECORD 10 ]--+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Internazionale
country_name    | Italy
array_to_string | 2005/2006, 2006/2007, 2007/2008, 2008/2009, 2009/2010
```
По английским клубам мы получаем 5 записей, вместо 7.

### Левостороннее (или правостороннее) соединение (Left/Right Join)

Левостороннее и правосторонее соединене не имеет почти никакой разницы. Единственное, что их отличает - какую таблицу мы называем "главной" и какую будем присоединять к ней.
В нашем случае будем смотреть пример левостороннего соединения - `left join`. Возьмем таблицу `clubs` и присоединим к ней таблицу с годами, когда эти клубы становились чемпионами:
```
joins=# select
joins-# clubs.name,
joins-# array_to_string(array_agg(DISTINCT year.year), ', ') as champions
joins-# from
joins-# clubs
joins-# left join year on year.id = clubs.champ_id
joins-# group by
joins-# clubs.name
joins-# order by
joins-# clubs.name LIMIT 10 \gx
-[ RECORD 1 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Arsenal
champions | 2001/2002, 2003/2004
-[ RECORD 2 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Atlético Madrid
champions | 2013/2014
-[ RECORD 3 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Barcelona
champions | 2004/2005, 2005/2006, 2008/2009, 2009/2010, 2010/2011, 2012/2013, 2014/2015, 2015/2016, 2017/2018, 2018/2019
-[ RECORD 4 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Bayer Leverkusen
champions |
-[ RECORD 5 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Bayern Munich
champions | 2000/2001, 2002/2003, 2004/2005, 2005/2006, 2007/2008, 2009/2010, 2012/2013, 2013/2014, 2014/2015, 2015/2016, 2016/2017, 2017/2018, 2018/2019, 2019/2020
-[ RECORD 6 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Borussia Dortmund
champions | 2001/2002, 2010/2011
-[ RECORD 7 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | CVfB Stuttgart
champions | 2006/2007
-[ RECORD 8 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Chelsea
champions | 2004/2005, 2005/2006, 2009/2010, 2014/2015, 2016/2017
-[ RECORD 9 ]-------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Deportivo La Coruña
champions |
-[ RECORD 10 ]------------------------------------------------------------------------------------------------------------------------------------------------------
name      | Everton
champions |
```
Как видим, левосторонее соединение позволило отобразить клубы, которые никогда не были чемпионами, чего не удавалось достигнуть при прямом соединении. Левосторонее соединение забирает все данные из левой таблицы, которые имеют связь с правой и выводит все данные из правой. 

### Полное соединение (Full Join)

Рассмотрим полное соединение на том же примере с командой и годами, когда она становилась чемпионом:
```
joins=# select
joins-# clubs.name as "Club",
joins-# array_to_string(array_agg(DISTINCT year.year), ', ') as "Champions"
joins-# from
joins-# clubs
joins-# full join year on year.id = clubs.champ_id
joins-# group by
joins-# clubs.name
joins-# ;
        Club         |                                                                        Champions
---------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------
 Arsenal             | 2001/2002, 2003/2004
 Atlético Madrid     | 2013/2014
 Barcelona           | 2004/2005, 2005/2006, 2008/2009, 2009/2010, 2010/2011, 2012/2013, 2014/2015, 2015/2016, 2017/2018, 2018/2019
 Bayer Leverkusen    |
 Bayern Munich       | 2000/2001, 2002/2003, 2004/2005, 2005/2006, 2007/2008, 2009/2010, 2012/2013, 2013/2014, 2014/2015, 2015/2016, 2016/2017, 2017/2018, 2018/2019, 2019/2020
 Borussia Dortmund   | 2001/2002, 2010/2011
 CVfB Stuttgart      | 2006/2007
 Chelsea             | 2004/2005, 2005/2006, 2009/2010, 2014/2015, 2016/2017
 Deportivo La Coruña |
 Everton             |
 Fiorentina          |
 Internazionale      | 2005/2006, 2006/2007, 2007/2008, 2008/2009, 2009/2010
 Juventus            | 2001/2002, 2002/2003, 2004/2005, 2011/2012, 2012/2013, 2013/2014, 2014/2015, 2015/2016, 2016/2017, 2017/2018, 2018/2019, 2019/2020
 Liverpool           | 2019/2020
 Manchecter United   | 2000/2001, 2002/2003, 2006/2007, 2007/2008, 2008/2009, 2010/2011, 2012/2013
 Manchester City     | 2011/2012, 2013/2014, 2017/2018, 2018/2019
 Milan               | 2003/2004, 2010/2011
 Napoli              |
 Parma               |
 RB Leipzig          |
 Real Betis          |
 Real Madrid         | 2000/2001, 2002/2003, 2006/2007, 2007/2008, 2011/2012, 2016/2017, 2019/2020
 Roma                | 2000/2001
 Schalke 04          |
 Sevilla             |
 Tottenham           |
 Valencia            | 2001/2002, 2003/2004
 VfL Wolfsburg       | 2008/2009
                     | 2020/2021
(29 rows)
```
При Full Join таблицы выводят все свои значения, в том числе и уникальные. Например, для `Schalke 04` нет данных в таблице `year`, также для сезона `2020/2021` нет чемпиона, так как он еще не завершился.

### Кросс соединение (Cross Join)

Кросс джлоин используюется тогда, когда каждое поле одной таблицы, необходимо соединить с каждым полем второй таблицы.
```
joins=# select distinct
joins-# clubs.name,
joins-# year.year
joins-# from
joins-# clubs
joins-# cross join year
joins-# order by  clubs.name, year.year LIMIT 30;
      name       |   year
-----------------+-----------
 Arsenal         | 2000/2001
 Arsenal         | 2001/2002
 Arsenal         | 2002/2003
 Arsenal         | 2003/2004
 Arsenal         | 2004/2005
 Arsenal         | 2005/2006
 Arsenal         | 2006/2007
 Arsenal         | 2007/2008
 Arsenal         | 2008/2009
 Arsenal         | 2009/2010
 Arsenal         | 2010/2011
 Arsenal         | 2011/2012
 Arsenal         | 2012/2013
 Arsenal         | 2013/2014
 Arsenal         | 2014/2015
 Arsenal         | 2015/2016
 Arsenal         | 2016/2017
 Arsenal         | 2017/2018
 Arsenal         | 2018/2019
 Arsenal         | 2019/2020
 Arsenal         | 2020/2021
 Atlético Madrid | 2000/2001
 Atlético Madrid | 2001/2002
 Atlético Madrid | 2002/2003
 Atlético Madrid | 2003/2004
 Atlético Madrid | 2004/2005
 Atlético Madrid | 2005/2006
 Atlético Madrid | 2006/2007
 Atlético Madrid | 2007/2008
 Atlético Madrid | 2008/2009
(30 rows)
```

### Соединение с несколькими типами

Чтобы получить актуальную информацию из наших таблиц по всем клубам, нам необходимо будет применить 2 типа соединения: `inner join` и `left join`.
```
joins=# select
joins-# clubs.name,
joins-# country.country_name,
joins-# array_to_string(array_agg(DISTINCT year.year), ', ')
joins-# from
joins-# clubs
joins-# join country on country.id = clubs.country_id
joins-# left join year on year.id = clubs.champ_id
joins-# group by
joins-# clubs.name,
joins-# country.country_name
joins-# order by
joins-# country.country_name,
joins-# clubs.name LIMIT 10 \gx
-[ RECORD 1 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Arsenal
country_name    | England
array_to_string | 2001/2002, 2003/2004
-[ RECORD 2 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Chelsea
country_name    | England
array_to_string | 2004/2005, 2005/2006, 2009/2010, 2014/2015, 2016/2017
-[ RECORD 3 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Everton
country_name    | England
array_to_string |
-[ RECORD 4 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Liverpool
country_name    | England
array_to_string | 2019/2020
-[ RECORD 5 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Manchecter United
country_name    | England
array_to_string | 2000/2001, 2002/2003, 2006/2007, 2007/2008, 2008/2009, 2010/2011, 2012/2013
-[ RECORD 6 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Manchester City
country_name    | England
array_to_string | 2011/2012, 2013/2014, 2017/2018, 2018/2019
-[ RECORD 7 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Tottenham
country_name    | England
array_to_string |
-[ RECORD 8 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Bayer Leverkusen
country_name    | Germany
array_to_string |
-[ RECORD 9 ]---+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Bayern Munich
country_name    | Germany
array_to_string | 2000/2001, 2002/2003, 2004/2005, 2005/2006, 2007/2008, 2009/2010, 2012/2013, 2013/2014, 2014/2015, 2015/2016, 2016/2017, 2017/2018, 2018/2019, 2019/2020
-[ RECORD 10 ]--+---------------------------------------------------------------------------------------------------------------------------------------------------------
name            | Borussia Dortmund
country_name    | Germany
array_to_string | 2001/2002, 2010/2011
```
Теперь нам доступны все команды и есть информация о том, кто из них становился чемпионом за последние 20 сезонов и сколько раз.