## Выполнение домашнего задания по теме "PostgreSQL и Azure, GCP, AWS"

### Вариант 2

### Установка PostgreSQL Server

При поиске готового решения PostgreSQL в GCP я наткнулся на PostgreSQL Server в GKE.
Мануал по установке из github оказался очень подробным и я решил установить его. Полный гайд находится по [ссылке](https://github.com/GoogleCloudPlatform/click-to-deploy/blob/master/k8s/postgresql/README.md).

### Подготовительные работы

1. Создаем кластер:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ export CLUSTER=postgresql-cluster
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ export ZONE=us-west1-a
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ gcloud container clusters create "$CLUSTER" --zone "$ZONE"
```
<pre><details><summary>Вывод терминала</summary>
WARNING: Starting in January 2021, clusters will use the Regular release channel by default when `--cluster-version`, `--release-channel`, `--no-enable-autoupgrade`, and `--no-enable-autorepair` flags are not specified.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting with version 1.18, clusters will have shielded GKE nodes by default.
WARNING: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s).
WARNING: Starting with version 1.19, newly created clusters and node-pools will have COS_CONTAINERD as the default node image when no image type is specified.
Creating cluster postgresql-cluster in us-west1-a... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1/projects/spartan-alcove-304312/zones/us-west1-a/clusters/postgresql-cluster].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-west1-a/postgresql-cluster?project=spartan-alcove-304312
kubeconfig entry generated for postgresql-cluster.
NAME                LOCATION    MASTER_VERSION   MASTER_IP       MACHINE_TYPE  NODE_VERSION     NUM_NODES  STATUS
postgresql-cluster  us-west1-a  1.17.15-gke.800  35.227.137.155  e2-medium     1.17.15-gke.800  3          RUNNING
</details></pre>

2. Настраиваем kubectl для подключению к кластеру:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
Fetching cluster endpoint and auth data.
kubeconfig entry generated for postgresql-cluster.
```
3. Клонируем репозиторий:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ git clone --recursive https://github.com/GoogleCloudPlatform/click-to-deploy.git
```
4. Устанавливаем компоненты приложения:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
customresourcedefinition.apiextensions.k8s.io/applications.app.k8s.io created
```


### Установка приложения

1. Переходим в директорию cd click-to-deploy/k8s/postgresql и устанавливаем переменные приложения:
<pre><details><summary>Вывод терминала</summary>
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ cd click-to-deploy/k8s/postgresql
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export APP_INSTANCE_NAME=postgresql-server
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export NAMESPACE=default
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export DEFAULT_STORAGE_CLASS="standard"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export PERSISTENT_DISK_SIZE="10Gi"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export TAG="9.6"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export IMAGE_POSTGRESQL="marketplace.gcr.io/google/postgresql"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ 
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export IMAGE_POSTGRESQL_EXPORTER="marketplace.gcr.io/google/postgresql/exporter:${TAG}"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export POSTGRESQL_DB_PASSWORD="$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 20 | head -n 1 | tr -d '\n')"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ 
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export EXPOSE_PUBLIC_SERVICE=false
</details></pre>

2. Создаем TLS сертификаты:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
>     -keyout /tmp/tls.key \
>     -out /tmp/tls.crt \
>     -subj "/CN=postgresql/O=postgresql"
Can't load /home/desmond/.rnd into RNG
140151647568320:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/desmond/.rnd
Generating a RSA private key
.............................................................................................................................................+++++
............+++++
writing new private key to '/tmp/tls.key'
-----
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export TLS_CERTIFICATE_KEY="$(cat /tmp/tls.key | base64)"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export TLS_CERTIFICATE_CRT="$(cat /tmp/tls.crt | base64)"
```
3. Создаем аккаунт:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ export POSTGRESQL_SERVICE_ACCOUNT="${APP_INSTANCE_NAME}-postgresql-sa"
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ kubectl create serviceaccount ${POSTGRESQL_SERVICE_ACCOUNT} --namespace ${NAMESPACE}
serviceaccount/postgresql-server-postgresql-sa created
```
4. Дополняем манифест переменными:

<pre><details><summary>Вывод терминала</summary>
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ helm template chart/postgresql \
>   --name-template "$APP_INSTANCE_NAME" \
>   --namespace "$NAMESPACE" \
>   --set postgresql.serviceAccount="$POSTGRESQL_SERVICE_ACCOUNT" \
>   --set postgresql.image.repo="$IMAGE_POSTGRESQL" \
>   --set postgresql.image.tag="$TAG" \
>   --set postgresql.exposePublicService="$EXPOSE_PUBLIC_SERVICE" \
>   --set postgresql.persistence.storageClass="${DEFAULT_STORAGE_CLASS}" \
>   --set postgresql.persistence.size="${PERSISTENT_DISK_SIZE}" \
>   --set db.password="$POSTGRESQL_DB_PASSWORD" \
>   --set metrics.image="$IMAGE_METRICS_EXPORTER" \
>   --set metrics.exporter.enabled="$METRICS_EXPORTER_ENABLED" \
>   --set exporter.image="$IMAGE_POSTGRESQL_EXPORTER" \
>   --set tls.base64EncodedPrivateKey="$TLS_CERTIFICATE_KEY" \
>   --set tls.base64EncodedCertificate="$TLS_CERTIFICATE_CRT" \
 > "$>   > "${APP_INSTANCE_NAME}_manifest.yaml"
</details></pre>

5. Применяем манифест:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ kubectl apply -f "${APP_INSTANCE_NAME}_manifest.yaml" --namespace "${NAMESPACE}"
secret/postgresql-server-secret created
secret/postgresql-server-tls created
service/postgresql-server-postgresql-svc created
service/postgresql-server-postgres-exporter-svc created
statefulset.apps/postgresql-server-postgresql created
application.app.k8s.io/postgresql-server created
```

6. Проверяем созданный кластер:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ echo "https://console.cloud.google.com/kubernetes/application/${ZONE}/${CLUSTER}/${NAMESPACE}/${APP_INSTANCE_NAME}"
https://console.cloud.google.com/kubernetes/application/us-west1-a/postgresql-cluster/default/postgresql-server
```
![Created](https://github.com/apovyshev/PostgreSQL/blob/main/15.CloudSQL/Created.PNG)

### Использование приложения

1. Пробросим порты и подключимся к базе данных:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev/click-to-deploy/k8s/postgresql$ kubectl port-forward \
 --names>   --namespace "${NAMESPACE}" \
>   "${APP_INSTANCE_NAME}-postgresql-0" 5432
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
```
В новом окне можно подключаться:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ psql -U postgres -h 127.0.0.1
psql (10.14 (Ubuntu 10.14-0ubuntu0.18.04.1), server 9.6.20)
SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=#
```
2. Проверка метрик. Для того чтобы получить метрики, необходимо получить IP адрес ноды кластера, где они лежат. Порт 9187:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl get all
NAME                                 READY   STATUS    RESTARTS   AGE
pod/postgresql-server-postgresql-0   2/2     Running   0          5m53s

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/kubernetes                                ClusterIP   10.23.240.1     <none>        443/TCP    30m
service/postgresql-server-postgres-exporter-svc   ClusterIP   10.23.254.157   <none>        9187/TCP   5m55s
service/postgresql-server-postgresql-svc          ClusterIP   10.23.245.190   <none>        5432/TCP   5m55s

NAME                                            READY   AGE
statefulset.apps/postgresql-server-postgresql   1/1     5m54s
```
Нам необходим IP 10.23.254.157

3. Пробрасываем порты 9187 для доступа из браузера по http://127.0.0.1:9187/metrics к метрикам:
```
desmond@BY1-WL-331:/mnt/c/Users/a_povyshev$ kubectl port-forward --namespace default postgresql-server-postgresql-0 9187
Forwarding from 127.0.0.1:9187 -> 9187
Forwarding from [::1]:9187 -> 9187
```
![Metrics](https://github.com/apovyshev/PostgreSQL/blob/main/15.CloudSQL/Metrics.PNG)
