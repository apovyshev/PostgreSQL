## Выполнение домашнего задания по второй теме "Установка PostgreSQL"
## 2 вариант:

### Создаем экземпляр VM в CGP
![VM3Created](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/2/VM3Created.PNG)

### Установка Docker Engine

1. По мануалу https://docs.docker.com/engine/install/ubuntu/ из официальной документации устанавливаем Docker Engine.
2. После установки для проверки работоспособности Docker запускаем простую команду `sudo docker run hello-world`:
```
desmond@postgres2:~$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:8c5aeeb6a5f3ba4883347d3747a7249f491766ca1caa47e5da5dfcf6b9b717c0
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
 ```

 ### Запуск контейнера docker с PostgreSQL 13

 1. Создадим отдельную папку для данных PostgreSQL нашего контейнера - `sudo mkdir /var/lib/postgres`.
 2. Так как мы создаем контейнер с базой данных и отдельно контейнер с клиентом необходимо создать для них общую сеть:
 ```
 desmond@postgres2:~$ sudo docker network create pg-net
ae888e62ce6de02fb83607b4e5e542ca4f53a0e77e9f7699a75feabd44b46818
```
3. Запускаем контейнер с PostgreSQL в нашей сети. Чтобы к контейнеру можно было подключиться из сети интернет, пробрасываем порт 5432 наружу, указав параметр `-d 5432:5432`:
```
desmond@postgres2:~$ sudo  docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
Unable to find image 'postgres:13' locally
13: Pulling from library/postgres
bb79b6b2107f: Pull complete
e3dc51fa2b56: Pull complete
f213b6f96d81: Pull complete
2780ac832fde: Pull complete
ae5cee1a3f12: Pull complete
95db3c06319e: Pull complete
475ca72764d5: Pull complete
8d602872ecae: Pull complete
cd38e50fcd6d: Pull complete
090a5b74af73: Pull complete
1a4356e41724: Pull complete
886a3b55e0f4: Pull complete
0aa963dfbf32: Pull complete
1403f0786df1: Pull complete
Digest: sha256:5344d1b6f33169b56f450c0586ffc3497f0479e1fc499bed76857912d2fe5f98
Status: Downloaded newer image for postgres:13
04c67e4b9a2e8e4ca8fc83f9daf0faaaaab493f8869b2f4b95dc79f6d48c80d7
```
4. Запускаем отдельный контейнер с клиентом:
```
desmond@postgres2:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-server -U postgres
Password for user postgres:
psql (13.1 (Debian 13.1-1.pgdg100+1))
Type "help" for help.

postgres=#
```
5. Создадим тестовые таблицы:
```
postgres=# create table test(c1 text);
sert into test values('1');
insert into test vCREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# insert into test values('2');
INSERT 0 1
postgres=# select * from test;
 c1
----
 1
 2
(2 rows)
```

### Подключение к базе данных PostgreSQL в контейнере виртуальной машины GCP со своей локальной машины

1. Для подключения к виртуальным машинам GCP необходимо добавить порт 5432 в Google VPC в брандмауэре:

![RuleCreated](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/2/RuleCreated.PNG)

2. Для подключения со своей локальной машины к базе данных на удаленном сервере необходимо установить клиент psql:
```
- sudo apt-get update
- sudo apt install postgresql-client -y
```
3. Подключаемся к базе используя публичный IP адрес сервера и проверяем данные нашей таблицы `test`:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ psql -h 34.71.132.65 -p 5432 -U postgres -W
Password for user postgres:
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 13.1 (Debian 13.1-1.pgdg100+1))
WARNING: psql major version 10, server major version 13.
         Some psql features might not work.
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
 2
(2 rows)

postgres=#
```
### Проверка целостности данных после пересоздания контейнеров
1. Для проверки, что наши данные не пропадают после удаления контейнеров, удалим их и запустим заново:
```
desmond@postgres2:~$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS
   PORTS                    NAMES
04c67e4b9a2e        postgres:13         "docker-entrypoint.s…"   32 minutes ago      Up 32 minutes       0.0.0.0:5432->5432/tcp   pg-server
desmond@postgres2:~$ sudo docker stop 04c67e4b9a2e
04c67e4b9a2e
desmond@postgres2:~$ sudo docker rm 04c67e4b9a2e
04c67e4b9a2e
desmond@postgres2:~$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
2. Запускаем контейнеры с данными и клиентом и проверяем наши данные:
```
desmond@postgres2:~$ sudo  docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
50afd066d0b81ae89d8571d1aec4304f2c3de5075bf158a97a732044ca7f823f
desmond@postgres2:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-server -U postgres
Password for user postgres:
psql (13.1 (Debian 13.1-1.pgdg100+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
 2
(2 rows)

postgres=#
```
Данные сохранились, так как при запуске контейнеров мы задали параметр `-v`, при котором все данные директории `/var/lib/postgresql/data` передавались в директории на хосте, где развернуты контейнеры - `/var/lib/postgres`.

