## Выполнение домашнего задания по теме "Разработка собственного приложения для работы с кластером"

Для своего приложения я выбрал кластер PostgreSQL от Crunchy Data. 
Приложение находится отдельно в репозитории https://github.com/apovyshev/psql_project

### Создание кластера GKE

1. Создание кластера GKE выполняем через `kubectl` консоли.
<pre><details><summary>Команда</summary>
gcloud beta container \
--project "spartan-alcove-304312" clusters create "postgres-ha" \
--zone "us-central1-c" \
--no-enable-basic-auth \
--cluster-version "1.18.15-gke.1501" \
--release-channel "regular" \
--machine-type "e2-medium" \
--image-type "COS" \
--disk-type "pd-ssd" \
--disk-size "100" \
--metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/cloud-platform" \
--num-nodes "3" \
--enable-stackdriver-kubernetes \
--enable-ip-alias \
--network "projects/spartan-alcove-304312/global/networks/default" \
--subnetwork "projects/spartan-alcove-304312/regions/us-central1/subnetworks/default" \
--default-max-pods-per-node "110" \
--no-enable-master-authorized-networks \
--addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
--enable-autoupgrade \
--enable-autorepair \
--max-surge-upgrade 1 \
--max-unavailable-upgrade 0 \
--enable-shielded-nodes \
--node-locations "us-central1-c"
</details></pre> 

2. Когда кластер будет, автоматически появится информация о нем:
```
NAME         LOCATION       MASTER_VERSION    MASTER_IP      MACHINE_TYPE  NODE_VERSION      NUM_NODES  STATUS
postgres-ha  us-central1-c  1.18.15-gke.1501  35.224.200.11  e2-medium     1.18.15-gke.1501  3          RUNNING
```

### Установка PGO Operator
Установку PGO будем выполнять по официальной документации [The PostgreSQL Operator Installer](https://access.crunchydata.com/documentation/postgres-operator/4.6.1/installation/postgres-operator/)
1. Для начала создаем отдельный неймспейс для кластера GKE:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl create namespace pgo
namespace/pgo created
```
2. Из официального репозитория скачиваем yaml файл с установкой оператора:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ wget https://raw.githubusercontent.com/CrunchyData/postgres-operator/v4.6.1/installers/kubectl/postgres-operator.yml
```
3. После того, как файл скачан, необходимо внести некоторые изменения, например, задать `pgo_admin_password` и заменить значения по умолчанию для `storage*_access_mode` на ReadWriteOnce, так как ReadWriteMany не поддерживается в GKE.

4. После внесения изменений применяем файл:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ kubectl apply -f postgres-operator.yml
serviceaccount/pgo-deployer-sa created
clusterrole.rbac.authorization.k8s.io/pgo-deployer-cr created
configmap/pgo-deployer-cm created
clusterrolebinding.rbac.authorization.k8s.io/pgo-deployer-crb created
job.batch/pgo-deploy created
```
5. Проверим результат командой `kubectl -n pgo get all`
<pre><details><summary>Вывод</summary>
NAME                                    READY   STATUS      RESTARTS   AGE
pod/pgo-deploy-k67tr                    0/1     Completed   0          47m
pod/postgres-operator-cf547b547-75vxz   4/4     Running     1          46m

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/postgres-operator   ClusterIP   10.108.11.122   <none>        8443/TCP,4171/TCP,4150/TCP   46m

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/postgres-operator   1/1     1            1           46m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/postgres-operator-cf547b547   1         1         1       47m

NAME                   COMPLETIONS   DURATION   AGE
job.batch/pgo-deploy   1/1           104s       48m
</details></pre> 


### Установка pgo client

1. Как в [инструкции](https://access.crunchydata.com/documentation/postgres-operator/4.6.1/installation/postgres-operator/) выполняем шаг Install the pgo Client:
<pre><details><summary>Вывод</summary>
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ curl https://raw.githubusercontent.com/CrunchyData/postgres-operator/v4.6.1/installers/kubectl/client-setup.sh > client-setup.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3516  100  3516    0     0   8451      0 --:--:-- --:--:-- --:--:--  8472
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ chmod +x client-setup.sh
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ ./client-setup.sh
pgo Client Binary detected at: /home/desmond/.pgo/pgo
Updating Binary...
Operating System found is Linux...
Downloading pgo version: v4.6.1...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   609  100   609    0     0   1471      0 --:--:-- --:--:-- --:--:--  1471
100 38.3M  100 38.3M    0     0  8723k      0  0:00:04  0:00:04 --:--:-- 10.0M

pgo client files have been generated, please add the following to your bashrc
export PATH=/home/desmond/.pgo/pgo:$PATH
export PGOUSER=/home/desmond/.pgo/pgo/pgouser
export PGO_CA_CERT=/home/desmond/.pgo/pgo/client.crt
export PGO_CLIENT_CERT=/home/desmond/.pgo/pgo/client.crt
export PGO_CLIENT_KEY=/home/desmond/.pgo/pgo/client.key
</details></pre>

2. В отдельном окне пробрасываем порт:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl -n pgo port-forward svc/postgres-operator 8443:8443
Forwarding from 127.0.0.1:8443 -> 8443
Forwarding from [::1]:8443 -> 8443
```
3. После того, как все шаги были выполнены четко по инструкции, пробуем проверить версию клиента:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo version
-bash: pgo: command not found
```
Получаем ошибку. Начинаем решать проблему..

4. Проверим местоположение исполняемого файла pgo:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ which pgo
```
Вывод ничего не дал.
5. Переносим файл из директории, куда он был скачан в `/usr/local/bin/` и снова проверяем версию:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ sudo mv /home/desmond/.pgo/pgo/pgo /usr/local/bin/
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo version
Error: /home/desmond/.pgo//pgouser file not found
```
Опять ошибка..
6. Заново пропишем переменные в `/.bashrc`:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ cat <<EOF >> ~/.bashrc
t PATH="> export PATH="$HOME/.pgo/$PGO_OPERATOR_NAMESPACE:$PATH"
> export PGOUSER="$HOME/.pgo/$PGO_OPERATOR_NAMESPACE/pgouser"
port PGO> export PGO_CA_CERT="$HOME/.pgo/$PGO_OPERATOR_NAMESPACE/client.crt"
> export PGO_CLIENT_CERT="$HOME/.pgo/$PGO_OPERATOR_NAMESPACE/client.crt"
> export PGO_CLIENT_KEY="$HOME/.pgo/$PGO_OPERATOR_NAMESPACE/client.key"
> EOF
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ source ${HOME?}/.bashrc
```
7. Проверяем клиент:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo version
pgo client version 4.6.1
pgo-apiserver version 4.6.1
```
Ну наконец-то. И еще один офф мануал, который работает криво :)

### Создание кластера PostgreSQL
1. Создаем кластер PostgreSQL через клиент pgo. При инициализации сразу укажем количество реплик, а также изменим тип сервиса на LoadBalancer:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo create -n pgo cluster persons --replica-count=3 --service-type=LoadBalancer
created cluster: persons
workflow id: d093dd4c-62d9-41a2-92f5-38287afe711e
database name: persons
users:
        username: testuser password: c3id8EJO+CfOX[_<(_lc-AdE
```
2. Проверим состояние кластера.
Через клиент pgo:
<pre><details><summary>desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo show -n pgo cluster persons</summary>
cluster : persons (crunchy-postgres-ha:centos8-13.2-4.6.1)
        pod : persons-8569469d44-mjfxt (Running) on gke-postgres-ha-default-pool-05adb2b7-7vs6 (1/1) (primary)
                pvc: persons (1Gi)
        pod : persons-iigs-667c576f5f-wwr7r (Running) on gke-postgres-ha-default-pool-05adb2b7-2dw2 (1/1) (replica)
                pvc: persons-iigs (1Gi)
        pod : persons-patn-84689974-rffhz (Running) on gke-postgres-ha-default-pool-05adb2b7-7vs6 (1/1) (replica)
                pvc: persons-patn (1Gi)
        pod : persons-scfb-7c5f7f7cc-mn8kx (Running) on gke-postgres-ha-default-pool-05adb2b7-jh91 (1/1) (replica)
                pvc: persons-scfb (1Gi)
        resources : Memory: 128Mi
        deployment : persons
        deployment : persons-backrest-shared-repo
        deployment : persons-iigs
        deployment : persons-patn
        deployment : persons-scfb
        service : persons - ClusterIP (10.108.15.77) ExternalIP (34.66.124.180) - Ports (2022/TCP, 5432/TCP)
        service : persons-replica - ClusterIP (10.108.10.119) ExternalIP (35.223.121.246) - Ports (2022/TCP, 5432/TCP)
        pgreplica : persons-iigs
        pgreplica : persons-patn
        pgreplica : persons-scfb
        labels : crunchy-pgha-scope=persons deployment-name=persons name=persons pg-cluster=persons pgo-version=4.6.1 pgouser=admin workflowid=d093dd4c-62d9-41a2-92f5-38287afe711e
</details></pre>

Через kubectl:
<pre><details><summary>desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl -n pgo get all</summary>
NAME                                              READY   STATUS      RESTARTS   AGE
pod/backrest-backup-persons-s9hmc                 0/1     Completed   0          82s
pod/persons-8569469d44-mjfxt                      1/1     Running     0          2m31s
pod/persons-backrest-shared-repo-cc95f5f6-mc8cm   1/1     Running     0          2m58s
pod/persons-iigs-667c576f5f-wwr7r                 1/1     Running     0          64s
pod/persons-patn-84689974-rffhz                   1/1     Running     0          63s
pod/persons-scfb-7c5f7f7cc-mn8kx                  1/1     Running     0          63s
pod/pgo-deploy-k67tr                              0/1     Completed   0          105m
pod/postgres-operator-cf547b547-75vxz             4/4     Running     1          104m

NAME                                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                         AGE
service/persons                        LoadBalancer   10.108.15.77    34.66.124.180    2022:30435/TCP,5432:32610/TCP   3m
service/persons-backrest-shared-repo   ClusterIP      10.108.12.247   <none>           2022/TCP                        3m
service/persons-replica                LoadBalancer   10.108.10.119   35.223.121.246   2022:30835/TCP,5432:31748/TCP   64s
service/postgres-operator              ClusterIP      10.108.11.122   <none>           8443/TCP,4171/TCP,4150/TCP      104m

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/persons                        1/1     1            1           3m1s
deployment.apps/persons-backrest-shared-repo   1/1     1            1           3m1s
deployment.apps/persons-iigs                   1/1     1            1           65s
deployment.apps/persons-patn                   1/1     1            1           65s
deployment.apps/persons-scfb                   1/1     1            1           64s
deployment.apps/postgres-operator              1/1     1            1           104m

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/persons-8569469d44                      1         1         1       3m
replicaset.apps/persons-backrest-shared-repo-cc95f5f6   1         1         1       3m
replicaset.apps/persons-iigs-667c576f5f                 1         1         1       65s
replicaset.apps/persons-patn-84689974                   1         1         1       65s
replicaset.apps/persons-scfb-7c5f7f7cc                  1         1         1       64s
replicaset.apps/postgres-operator-cf547b547             1         1         1       104m

NAME                                COMPLETIONS   DURATION   AGE
job.batch/backrest-backup-persons   1/1           18s        83s
job.batch/pgo-deploy                1/1           104s       105m
</details></pre>

Как видим, при указании реплик и сервиса LoadBalancer у нас создалось 2 балансера. При этом доступ к базе работает одинаково как с одного внешнего IP, так и со второго.

3. Пробуем подключиться к базе и создать таблицу для нашего приложения:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ psql -h 34.66.124.180 -U testuser -d persons -W
Password for user testuser:
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 13.2)
WARNING: psql major version 10, server major version 13.
         Some psql features might not work.
Type "help" for help.

persons=>
```
В качестве пользователя, базы и пароля используем те данные, которые нам сченерировал клиент pgo при запуске кластера.

<pre><details><summary>Вывод создания таблицы и ее наполнения</summary>
persons=> CREATE TABLE persons (
persons(>   ID SERIAL PRIMARY KEY,
persons(>   name VARCHAR,
persons(>   age INTEGER
persons(> );
CREATE TABLE
persons=> INSERT INTO persons (name, age) VALUES ('Tyler', 33);
ERT INTO persons (name, age) VALUES ('Natasha', 33);
INSERT INTO persons (name, age) VALUES ('Dima', 33);
INSERT INTO persons (name, age) VALUES ('Sonya', 33);
INSERT INTO persons (name, age) VALUES ('Timo', 33);
INSERT INTO persons (name, age) VALUES ('Nastya', 33);
INSERT INTO persons (name, age) VALUES ('Alex', 33);
INSERT INTO persons (name, age) VALUES ('Zina', 33);
INSERT INTO persons (name, age) VALUES ('Petr', 33);
INSERT INTO persons (name, age) VALUES ('Egor', 33);INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Natasha', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Dima', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Sonya', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Timo', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Nastya', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Alex', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Zina', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Petr', 33);
INSERT 0 1
persons=> INSERT INTO persons (name, age) VALUES ('Egor', 33);
INSERT 0 1
</details></pre>

3. Удалим одну реплику и подключимся с другого IP балансера для проверки данных:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo -n pgo scaledown persons --target=persons-iigs
WARNING: Are you sure? (yes/no): yes
deleted replica persons-iigs
```
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ psql -h 35.223.121.246 -U testuser -d persons -W
Password for user testuser:
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 13.2)
WARNING: psql major version 10, server major version 13.
         Some psql features might not work.
Type "help" for help.

persons=> select * from persons;
 id |  name   | age
----+---------+-----
  1 | Tyler   |  33
  2 | Natasha |  33
  3 | Dima    |  33
  4 | Sonya   |  33
  5 | Timo    |  33
  6 | Nastya  |  33
  7 | Alex    |  33
  8 | Zina    |  33
  9 | Petr    |  33
 10 | Egor    |  33
(10 rows)

persons=>
```
Подключение прошло успешно, все данные есть, можно подключать приложение, но прежде вернем реплику обратно:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/crunchy$ pgo scale -n pgo persons --replica-count=1
WARNING: Are you sure? (yes/no): yes
created Pgreplica persons-dpcn
```


### Настройка приложения для коннекта к базе данных PostgreSQL

1. Для того, чтобы связать наше приложение с кластером, необходимо внести информацию в `queries.js`:
```
---
const pool = new Pool({
	  user: 'testuser',
	  host: '34.66.124.180',
	  database: 'persons',
	  password: 'c3id8EJO+CfOX[_<(_lc-AdE',
	  port: 5432,
})
---
```
2. Сохраняем файл и запускаем приложение:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/psql_project/app$ node index.js
App running on port 3000.
```
3. Пробуем выполнить несколько тестов:
- Проверим запуск приложения, в браузере пройдем по ссылке http://localhost:3000

![Launched](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/Launched.PNG)

- Проверим всех пльзователей, в браузере пройдем по ссылке http://localhost:3000/persons

![All](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/All.PNG)

- Проверим всех пльзователей по id, в браузере пройдем по ссылке http://localhost:3000/persons/2

![id](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/id.PNG)

- Проверим добавление нового пользователя, в отдельном окне терминала выполним:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ curl --data "name=Sasha&age=40" http://localhost:3000/persons
User added with ID: 34
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ curl --data "name=Sasha&age=40" http://localhost:3000/persons
User added with ID: 35
```
![post](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/post.PNG)
```
persons=> select * from persons;
 id |  name   | age
----+---------+-----
  1 | Tyler   |  33
  2 | Natasha |  33
  3 | Dima    |  33
  4 | Sonya   |  33
  5 | Timo    |  33
  6 | Nastya  |  33
  7 | Alex    |  33
  8 | Zina    |  33
  9 | Petr    |  33
 10 | Egor    |  33
 34 | Sasha   |  40
 35 | Sasha   |  40
(12 rows)
```
- Проверим добавление обновление пользователя, в отдельном окне терминала выполним:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ curl -X PUT --data "name=Gosha&age=33" http://localhost:3000/persons/35
User modified with ID: 35
```
![put](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/put.PNG)
```
persons=> select * from persons where id=35;
 id | name  | age
----+-------+-----
 35 | Gosha |  33
(1 row)

persons=>
```
- Проверим удаление пользователя, в отдельном окне терминала выполним:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ curl -X DELETE http://localhost:3000/persons/35
User deleted with ID: 35
```
![del](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/del.PNG)
```
persons=> select * from persons where id=35;
 id | name | age
----+------+-----
(0 rows)

persons=>
```
Приложение работает, база данных обновляется.

### Установка приложения в GKE
После того, как были внесены правки в queries.js файл, я запушил новый код в github в репозиторий https://github.com/apovyshev/psql_project . В dockerhub у меня тоже есть проект https://hub.docker.com/repository/docker/apovyshev/postgres/general , который настроен на github через хук. При обновлении кода, начинает билдиться новый образ, который мы в последствии будем брать для развертывания приложения в GKE.

Собственно, свежий билд уже готов:
![docker](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/docker.PNG)

1. После того, как мы убедились, что приложение и база работает, что новый билд собран в dockerhub, поместим приложение в GKE:
<pre><details><summary>kube_app.yaml</summary>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: persons-app-deployment
spec:
  selector:
    matchLabels:
      app: persons-app
  replicas: 1
  template:
    metadata:
      labels:
        app: persons-app
    spec:
      containers:
      - name: persons
        image: apovyshev/postgres:app
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: persons-loadbalancer
  labels:
    app: persons-app
spec:
  type: LoadBalancer
  ports:
    - port: 3000
      targetPort: 3000
      name: http 
  selector:
    app: persons-app   
</details></pre>
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/psql_project$ kubectl -n pgo create -f kube_app.yaml
```
2. Проверим наши поды:
<pre><details><summary>Вывод kubectl -n pgo get all</summary>
NAME                                              READY   STATUS      RESTARTS   AGE
pod/persons-8569469d44-mjfxt                      1/1     Running     0          85m
pod/persons-app-deployment-5648b44d8c-rmllg       1/1     Running     0          16m
pod/persons-backrest-shared-repo-cc95f5f6-mc8cm   1/1     Running     0          85m
pod/persons-dpcn-65c65f95d-tqgrh                  1/1     Running     0          57m
pod/persons-patn-84689974-rffhz                   1/1     Running     0          83m
pod/persons-scfb-7c5f7f7cc-mn8kx                  1/1     Running     0          83m
pod/pgo-deploy-k67tr                              0/1     Completed   0          3h8m
pod/postgres-operator-cf547b547-75vxz             4/4     Running     1          3h7m

NAME                                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                         AGE
service/persons                        LoadBalancer   10.108.15.77    34.66.124.180    2022:30435/TCP,5432:32610/TCP   85m
service/persons-backrest-shared-repo   ClusterIP      10.108.12.247   <none>           2022/TCP                        85m
service/persons-loadbalancer           LoadBalancer   10.108.12.103   35.238.122.96    3000:32666/TCP                  16m
service/persons-replica                LoadBalancer   10.108.10.119   35.223.121.246   2022:30835/TCP,5432:31748/TCP   83m
service/postgres-operator              ClusterIP      10.108.11.122   <none>           8443/TCP,4171/TCP,4150/TCP      3h7m

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/persons                        1/1     1            1           85m
deployment.apps/persons-app-deployment         1/1     1            1           16m
deployment.apps/persons-backrest-shared-repo   1/1     1            1           85m
deployment.apps/persons-dpcn                   1/1     1            1           57m
deployment.apps/persons-patn                   1/1     1            1           83m
deployment.apps/persons-scfb                   1/1     1            1           83m
deployment.apps/postgres-operator              1/1     1            1           3h7m

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/persons-8569469d44                      1         1         1       85m
replicaset.apps/persons-app-deployment-5648b44d8c       1         1         1       16m
replicaset.apps/persons-backrest-shared-repo-cc95f5f6   1         1         1       85m
replicaset.apps/persons-dpcn-65c65f95d                  1         1         1       57m
replicaset.apps/persons-patn-84689974                   1         1         1       83m
replicaset.apps/persons-scfb-7c5f7f7cc                  1         1         1       83m
replicaset.apps/postgres-operator-cf547b547             1         1         1       3h7m

NAME                   COMPLETIONS   DURATION   AGE
job.batch/pgo-deploy   1/1           104s       3h8m
</details></pre>
Наше приложение в кубе и доступно по 35.238.122.96

3. Теперь наше приложение доступно через ip 35.238.122.96:

![app](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/app.PNG)

### Тестирование приложения при помощи yandex-tank

Yandex-tank будем использовать через docker. 
1. Указываем IP адрес нашего приложения, генерируем токен для веб-версии танка.
<pre><details><summary>Файл yandex-tank</summary>
overload:
  enabled: true
  package: yandextank.plugins.DataUploader
  token_file: "token.txt"
phantom:
  address: 35.238.122.96:3000
  instances: 50
  load_profile:
    load_type: rps 
    schedule: line(100, 1000, 5m)
  header_http: "1.1"
  headers:
    - "[Host: 35.238.122.96:3000]"
    - "[Connection: keep-alive]"
    - "[User-Agent: Tank]"
    - "[Content-Type: application/json]"
  uris:
    - "/persons/10"
    - "/persons"
console:
  enabled: true
telegraf:
  enabled: false
</details></pre>
2. Запускаем нагрузку:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/psql_project$ docker run -v $(pwd):/var/loadtest --rm --net host -it direvius/yandex-tank
```
3. Результаты:

Вывод консоли

![yandex](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/yandex.PNG)

Информация из WEB

![yandex-web](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/yandex-web.PNG)

![yandex-web1](https://github.com/apovyshev/PostgreSQL/blob/main/16.APIapp/yandex-web1.PNG)

Приложение справляется с нагрузками, все работает.