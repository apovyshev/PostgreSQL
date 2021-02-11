## Выполнение домашнего задания по теме "Работа c PostgreSQL в Kubernetes"

### Подготовка инфраструктуры

Создаем кластер GKE через веб.
![ClusterCreated](https://github.com/apovyshev/PostgreSQL/blob/main/14.K8s/ClusterCreated.PNG)

```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ gcloud container clusters list
NAME   LOCATION       MASTER_VERSION    MASTER_IP     MACHINE_TYPE  NODE_VERSION      NUM_NODES  STATUS
citus  us-central1-c  1.17.14-gke.1600  34.70.46.183  e2-medium     1.17.14-gke.1600  3          RUNNING
```

### Разворачивание CitusDB

1. Клонируем репозиторий https://github.com/jinhong-/citus-k8s себе на локальную машину.
2. Первым делом создаем secret:

<pre><details><summary>Содержимое secrets.yaml</summary>
apiVersion: v1
kind: Secret
metadata:
  name: citus-secrets
type: Opaque
data:
  password: b3R1czMyMSQ=
</details></pre>

```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ kubectl create -f secrets.yaml
secret/citus-secrets created
```
3. Создаем `persistentvolumeclaim`, `service` и `deployment.apps`:

<pre><details><summary>Содержимое master.yaml</summary>

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: citus-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
---
apiVersion: v1
kind: Service
metadata:
  name: citus-master
  labels:
    app: citus-master
spec:
  selector:
    app: citus-master
  type: LoadBalancer
  ports:
   - port: 5432
     targetPort: 5432
     protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: citus-master
spec:
  selector:
    matchLabels:
      app: citus-master
  replicas: 1
  template:
    metadata:
      labels:
        app: citus-master
    spec:
      containers:
      - name: citus
        image: citusdata/citus:7.3.0
        ports:
        - containerPort: 5432
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: citus-master-pvc
</details></pre>

```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ kubectl create -f master.yaml
persistentvolumeclaim/citus-master-pvc created
service/citus-master created
deployment.apps/citus-master created
```
3. Создаем воркеров:

<pre><details><summary>Содержимое workers.yaml</summary>
apiVersion: v1
kind: Service
metadata:
  name: citus-workers
  labels:
    app: citus-workers
spec:
  selector:
    app: citus-workers
  clusterIP: None
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: citus-worker
spec:
  selector:
    matchLabels:
      app: citus-workers
  serviceName: citus-workers
  replicas: 5
  template:
    metadata:
      labels:
        app: citus-workers
    spec:
      containers:
      - name: citus-worker
        image: citusdata/citus:7.3.0
        lifecycle:
          postStart:
            exec:
              command: 
              - /bin/sh
              - -c
              - if [ ${POD_IP} ]; then psql --host=citus-master --username=postgres --command="SELECT * from master_add_node('${HOSTNAME}.citus-workers', 5432);" ; fi
        ports:
        - containerPort: 5432
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: citus-secrets
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - ./pg_healthcheck
          initialDelaySeconds: 60
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
</details></pre>

```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ kubectl create -f workers.yaml
service/citus-workers created
statefulset.apps/citus-worker created
```

4. Проверяем состояние кластера:

<pre><details><summary>desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ kubectl get all</summary>
NAME                                READY   STATUS    RESTARTS   AGE
pod/citus-master-74fb74646c-2w5sc   1/1     Running   0          79m
pod/citus-worker-0                  1/1     Running   0          78m
pod/citus-worker-1                  1/1     Running   0          78m
pod/citus-worker-2                  1/1     Running   0          77m
pod/citus-worker-3                  1/1     Running   0          77m
pod/citus-worker-4                  1/1     Running   0          77m

NAME                    TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)          AGE
service/citus-master    LoadBalancer   10.80.9.22   35.184.67.212   5432:30974/TCP   79m
service/citus-workers   ClusterIP      None         <none>          5432/TCP         78m
service/kubernetes      ClusterIP      10.80.0.1    <none>          443/TCP          85m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/citus-master   1/1     1            1           79m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/citus-master-74fb74646c   1         1         1       79m

NAME                            READY   AGE
statefulset.apps/citus-worker   5/5     78m
</details></pre>

### Импорт данных из GCS в K8s

1. Подключаемся к мастеру нашего кластера:

```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ kubectl exec -it pod/citus-master-74fb74646c-2w5sc -- bash
root@citus-master-74fb74646c-2w5sc:/#
```
2. Согласно [инструкции](https://cloud.google.com/sdk/docs/quickstart#deb) устанавливаем `Cloud SDK`.
3. Создаем новую директорию в tmp и копируем туда файлы из GCS:
<pre><details><summary>root@citus-master-74fb74646c-2w5sc:/tmp/taxi# gsutil -m cp \</summary>
>    "gs://taxi20210211/taxi_trips000000000000.csv" \
>    "gs://taxi20210211/taxi_trips000000000001.csv" \
>    "gs://taxi20210211/taxi_trips000000000002.csv" \
>    "gs://taxi20210211/taxi_trips000000000003.csv" \
>    "gs://taxi20210211/taxi_trips000000000004.csv" \
>    "gs://taxi20210211/taxi_trips000000000005.csv" \
>    "gs://taxi20210211/taxi_trips000000000006.csv" \
>    "gs://taxi20210211/taxi_trips000000000007.csv" \
>    "gs://taxi20210211/taxi_trips000000000008.csv" \
>    "gs://taxi20210211/taxi_trips000000000009.csv" \
>    "gs://taxi20210211/taxi_trips000000000010.csv" \
>    "gs://taxi20210211/taxi_trips000000000011.csv" \
>    "gs://taxi20210211/taxi_trips000000000012.csv" \
>    "gs://taxi20210211/taxi_trips000000000013.csv" \
>    "gs://taxi20210211/taxi_trips000000000014.csv" \
>    "gs://taxi20210211/taxi_trips000000000015.csv" \
>    "gs://taxi20210211/taxi_trips000000000016.csv" \
>    "gs://taxi20210211/taxi_trips000000000017.csv" \
>    "gs://taxi20210211/taxi_trips000000000018.csv" \
>    "gs://taxi20210211/taxi_trips000000000019.csv" \
>    "gs://taxi20210211/taxi_trips000000000020.csv" \
>    "gs://taxi20210211/taxi_trips000000000021.csv" \
>    "gs://taxi20210211/taxi_trips000000000022.csv" \
>    "gs://taxi20210211/taxi_trips000000000023.csv" \
>    "gs://taxi20210211/taxi_trips000000000024.csv" \
>    "gs://taxi20210211/taxi_trips000000000025.csv" \
>    "gs://taxi20210211/taxi_trips000000000026.csv" \
>    "gs://taxi20210211/taxi_trips000000000027.csv" \
>    "gs://taxi20210211/taxi_trips000000000028.csv" \
>    "gs://taxi20210211/taxi_trips000000000029.csv" \
>    "gs://taxi20210211/taxi_trips000000000030.csv" \
>    "gs://taxi20210211/taxi_trips000000000031.csv" \
>    "gs://taxi20210211/taxi_trips000000000032.csv" \
>    "gs://taxi20210211/taxi_trips000000000033.csv" \
>    "gs://taxi20210211/taxi_trips000000000034.csv" \
>    "gs://taxi20210211/taxi_trips000000000035.csv" \
>    "gs://taxi20210211/taxi_trips000000000036.csv" \
>    "gs://taxi20210211/taxi_trips000000000037.csv" \
>    "gs://taxi20210211/taxi_trips000000000038.csv" \
>    "gs://taxi20210211/taxi_trips000000000039.csv" \
>    "gs://taxi20210211/taxi_trips000000000040.csv" \
>    .
Copying gs://taxi20210211/taxi_trips000000000000.csv...
Copying gs://taxi20210211/taxi_trips000000000001.csv...
Copying gs://taxi20210211/taxi_trips000000000002.csv...
Copying gs://taxi20210211/taxi_trips000000000003.csv...
Copying gs://taxi20210211/taxi_trips000000000004.csv...
Copying gs://taxi20210211/taxi_trips000000000005.csv...
Copying gs://taxi20210211/taxi_trips000000000006.csv...
Copying gs://taxi20210211/taxi_trips000000000007.csv...
Copying gs://taxi20210211/taxi_trips000000000008.csv...
Copying gs://taxi20210211/taxi_trips000000000009.csv...
Copying gs://taxi20210211/taxi_trips000000000010.csv.../s ETA 00:02:14
Copying gs://taxi20210211/taxi_trips000000000011.csv.../s ETA 00:02:20
Copying gs://taxi20210211/taxi_trips000000000012.csv.../s ETA 00:02:21
Copying gs://taxi20210211/taxi_trips000000000013.csv.../s ETA 00:02:23
Copying gs://taxi20210211/taxi_trips000000000014.csv.../s ETA 00:02:31
Copying gs://taxi20210211/taxi_trips000000000015.csv.../s ETA 00:08:27
Copying gs://taxi20210211/taxi_trips000000000016.csv.../s ETA 00:08:28
Copying gs://taxi20210211/taxi_trips000000000017.csv.../s ETA 00:08:24
Copying gs://taxi20210211/taxi_trips000000000018.csv.../s ETA 00:08:02
Copying gs://taxi20210211/taxi_trips000000000019.csv...B/s ETA 00:07:09
Copying gs://taxi20210211/taxi_trips000000000020.csv...B/s ETA 00:04:37
Copying gs://taxi20210211/taxi_trips000000000021.csv...B/s ETA 00:04:41
Copying gs://taxi20210211/taxi_trips000000000022.csv...B/s ETA 00:04:36
Copying gs://taxi20210211/taxi_trips000000000023.csv...B/s ETA 00:04:44
Copying gs://taxi20210211/taxi_trips000000000024.csv...B/s ETA 00:04:25
Copying gs://taxi20210211/taxi_trips000000000025.csv...B/s ETA 00:03:39
Copying gs://taxi20210211/taxi_trips000000000026.csv...B/s ETA 00:03:43
Copying gs://taxi20210211/taxi_trips000000000027.csv...B/s ETA 00:03:42
Copying gs://taxi20210211/taxi_trips000000000028.csv...B/s ETA 00:03:41
Copying gs://taxi20210211/taxi_trips000000000029.csv...B/s ETA 00:03:39
Copying gs://taxi20210211/taxi_trips000000000030.csv...B/s ETA 00:02:50
Copying gs://taxi20210211/taxi_trips000000000031.csv...B/s ETA 00:02:50
Copying gs://taxi20210211/taxi_trips000000000032.csv...B/s ETA 00:02:50
Copying gs://taxi20210211/taxi_trips000000000033.csv...B/s ETA 00:02:45
Copying gs://taxi20210211/taxi_trips000000000034.csv...B/s ETA 00:02:48
Copying gs://taxi20210211/taxi_trips000000000035.csv...B/s ETA 00:01:58
Copying gs://taxi20210211/taxi_trips000000000036.csv...B/s ETA 00:01:58
Copying gs://taxi20210211/taxi_trips000000000037.csv...B/s ETA 00:01:58
Copying gs://taxi20210211/taxi_trips000000000038.csv...B/s ETA 00:01:53
Copying gs://taxi20210211/taxi_trips000000000039.csv...B/s ETA 00:01:52
Copying gs://taxi20210211/taxi_trips000000000040.csv...B/s ETA 00:01:06
- [41/41 files][ 10.2 GiB/ 10.2 GiB] 100% Done  27.0 MiB/s ETA 00:00:00
Operation completed over 41 objects/10.2 GiB. 
</details></pre>

4. После того, как скачали файлы, копируем их в созданную таблицу:
```
root@citus-master-74fb74646c-2w5sc:/# psql -U postgres
psql (10.3 (Debian 10.3-1.pgdg90+1))
Type "help" for help.

postgres=# \c taxi
You are now connected to database "taxi" as user "postgres".
taxi=#
```
<pre><details><summary>Вывод копирования данных</summary>
taxi=# create table taxi_trips (
unique_key text,
taxi_id text,
trip_start_timestamp TIMESTAMP,
trip_end_timestamp TIMESTAMP,
trip_seconds bigint,
trip_miles numeric,
pickup_census_tract bigint,
dropoff_census_tract bigint,
pickup_community_area bigint,
dropoff_community_area bigint,
fare numeric,
tips numeric,
tolls numeric,
extras numeric,
trip_total numeric,
payment_type text,
company text,
pickup_latitude numeric,
pickup_longitude numeric,
pickup_location text,
dropoff_latitude numeric,
dropoff_longitude numeric,
dropoff_location text
);
CREATE TABLE
taxi=# COPY taxi_trips(unique_key,
taxi_id,
trip_start_timestamp,
trip_end_timestamp,
trip_seconds,
trip_miles,
pickup_census_tract,
dropoff_census_tract,
pickup_community_area,
dropoff_community_area,
fare,
tips,
tolls,
extras,
trip_total,
payment_type,
company,
pickup_latitude,
pickup_longitude,
pickup_location,
dropoff_latitude,
dropoff_longitude,
dropoff_location)
FROM PROGRAM 'awk FNR-1 /tmp/taxi/*.csv | cat' with (format csv, delimiter ',', header);
COPY 28018816
</details></pre>

### Тестирование производительности

1. После того, как все данные были залиты на сервер, можем подключиться к базе с локального хоста и приступить к тестированию:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/pg25/citus_LB$ psql -h 35.184.67.212 -U postgres --password -p 5432
Password for user postgres:
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 10.3 (Debian 10.3-1.pgdg90+1))
Type "help" for help.

postgres=#
```
2. Подключимся к базе данных и выполним запрос:
```
postgres=# \c taxi
Password for user postgres:
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 10.3 (Debian 10.3-1.pgdg90+1))
You are now connected to database "taxi" as user "postgres".
taxi=# \timing
Timing is on.
taxi=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(*) as c
M taxi_taxi-# FROM taxi_trips
taxi-# group by payment_type
taxi-# order by 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Prepaid      |            0 |       59
 Way2ride     |           15 |       95
 Split        |           18 |      333
 Pcard        |            3 |     6790
 Dispute      |            0 |    13078
 Mobile       |           15 |    29997
 Prcard       |            1 |    70525
 Unknown      |            1 |    82550
 No Charge    |            6 |   144959
 Credit Card  |           17 | 12292156
 Cash         |            0 | 15378274
(11 rows)

Time: 192276.789 ms (03:12.277)
```
Запрос выполнился за 03:12.277.

3. Для сравнения выполним такой же запрос в PostgreSQL 13:
```
postgres=# SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) as tips_percent, count(*) as c
postgres-# FROM taxi_trips
postgres-# group by payment_type
postgres-# order by 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Prepaid      |            0 |       59
 Way2ride     |           15 |       95
 Split        |           18 |      333
 Pcard        |            3 |     6790
 Dispute      |            0 |    13078
 Mobile       |           15 |    29997
 Prcard       |            1 |    70525
 Unknown      |            1 |    82550
 No Charge    |            6 |   144959
 Credit Card  |           17 | 12292156
 Cash         |            0 | 15378274
(11 rows)

Time: 561741.684 ms (09:21.742)
```
Как видим, разница в 6 с небольшим минут.