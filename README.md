# Домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных. Его можно найти в папке 13-kubernetes-config.

<details>

  <summary>Описание задачи</summary>  
## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 2 контейнера — фронтенд, бекенд;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.

</details>

### Решение

1. Создаем namespace и pv для stage контура:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-stage
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /data/pv-stage
---
apiVersion: v1
kind: Namespace
metadata:
  name: stage
```
2. Запускаем

```bash
kubectl apply -f prepare-stage.yaml 

persistentvolume/pv-stage created
namespace/stage created
```

3. Подготовим и применим два манифеста для приложений (фронт и бэк в одном поде), и базы данных (в отдельном поде).

Комментарии указаны в манифестах:
stage-app.yaml
stage-db.yaml


```bash
kubectl apply -f ./stage/
deployment.apps/stage created
service/stage-app-srv created
service/stage-db-srv created
statefulset.apps/db created
```

4. Проверяем что все компоненты развернулись корректно
```bash
kubectl get deployment -n stage
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
stage   1/1     1            1           4m53s

kubectl get pods -n stage -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
db-0                     1/1     Running   0          5m34s   10.233.90.41   node1   <none>           <none>
stage-5b6f945bcb-q66ks   2/2     Running   0          5m34s   10.233.94.39   node0   <none>           <none>

kubectl get services -n stage -o wide
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE     SELECTOR
stage-app-srv   ClusterIP   10.233.50.180   <none>        9000/TCP,8080/TCP   6m22s   app=stage
stage-db-srv    ClusterIP   10.233.38.11    <none>        5432/TCP            6m22s   app=postgres
```

5. Прверяем логи контейнеров в поде

```
kubectl logs -n stage stage-5b6f945bcb-q66ks back-stage
INFO:     Uvicorn running on http://0.0.0.0:9000 (Press CTRL+C to quit)
INFO:     Started reloader process [7] using statreload
INFO:     Started server process [9]
INFO:     Waiting for application startup.
INFO:     Application startup complete.

kubectl logs -n stage stage-5b6f945bcb-q66ks front-stage
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: /etc/nginx/conf.d/default.conf differs from the packaged version
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/04/23 10:16:31 [notice] 1#1: using the "epoll" event method
2022/04/23 10:16:31 [notice] 1#1: nginx/1.21.6
2022/04/23 10:16:31 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/04/23 10:16:31 [notice] 1#1: OS: Linux 5.4.0-42-generic
2022/04/23 10:16:31 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/04/23 10:16:31 [notice] 1#1: start worker processes
2022/04/23 10:16:31 [notice] 1#1: start worker process 30
2022/04/23 10:16:31 [notice] 1#1: start worker process 31

kubectl logs -n stage db-0 | tail -n 6
2022-04-23 10:16:32.309 UTC [1] LOG:  starting PostgreSQL 13.6 on x86_64-pc-linux-musl, compiled by gcc (Alpine 10.3.1_git20211027) 10.3.1 20211027, 64-bit
2022-04-23 10:16:32.310 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-04-23 10:16:32.310 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2022-04-23 10:16:32.324 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-04-23 10:16:32.345 UTC [51] LOG:  database system was shut down at 2022-04-23 10:16:32 UTC
2022-04-23 10:16:32.350 UTC [1] LOG:  database system is ready to accept connections

```

6. При необходимости можно сделать форвард портов и проверить работоспособность api и интерфейс фронта.

---


<details>

  <summary>Описание задачи</summary>  
## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

</details>

### Решение
1. Создаем namespace и pv для prod контура:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-prod
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /data/pv-prod
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod

```

2. Запускаем

```bash
kubectl apply -f prepare-prod.yaml 

persistentvolume/pv-prod created
namespace/prod created
```

3. Подготовим и применим два манифеста для приложений (фронт и бэк в одном поде), и базы данных (в отдельном поде).

Комментарии указаны в манифестах:
prod-back.yaml
prod-front.yaml
prod-db.yaml

```bash
kubectl apply -f ./prod/
deployment.apps/back created
service/prod-back-srv created
service/prod-db-srv created
statefulset.apps/db created
deployment.apps/front created
service/prod-front-srv created
```
4. Проверяем что все компоненты развернулись корректно
```bash
kubectl get deployment -n prod
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
back    1/1     1            1           115s
front   1/1     1            1           115s

kubectl get pods -n prod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
back-8557cb55c9-qfzqt    1/1     Running   0          2m14s   10.233.94.54   node0   <none>           <none>
db-0                     1/1     Running   0          2m14s   10.233.94.55   node0   <none>           <none>
front-6d5bc4d889-k8c95   1/1     Running   0          2m14s   10.233.94.56   node0   <none>           <none>

kubectl get services -n prod -o wide
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
prod-back-srv    ClusterIP   10.233.30.143   <none>        9000/TCP   2m30s   app=back
prod-db-srv      ClusterIP   10.233.45.153   <none>        5432/TCP   2m30s   app=postgres
prod-front-srv   ClusterIP   10.233.61.144   <none>        8080/TCP   2m30s   app=front
```

5. Прверяем логи контейнеров в подах

```
kubectl logs -n prod back-8557cb55c9-qfzqt 
INFO:     Uvicorn running on http://0.0.0.0:9000 (Press CTRL+C to quit)
INFO:     Started reloader process [7] using statreload
INFO:     Started server process [9]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     127.0.0.1:41202 - "GET / HTTP/1.1" 404 Not Found
INFO:     127.0.0.1:41202 - "GET /favicon.ico HTTP/1.1" 404 Not Found
INFO:     127.0.0.1:41202 - "GET /news HTTP/1.1" 404 Not Found
INFO:     127.0.0.1:41322 - "GET /api/news HTTP/1.1" 307 Temporary Redirect
INFO:     127.0.0.1:41322 - "GET /api/news/ HTTP/1.1" 200 OK

kubectl logs -n prod front-6d5bc4d889-k8c95 | tail -n 6
2022/04/23 11:41:22 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/04/23 11:41:22 [notice] 1#1: OS: Linux 5.4.0-42-generic
2022/04/23 11:41:22 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/04/23 11:41:22 [notice] 1#1: start worker processes
2022/04/23 11:41:22 [notice] 1#1: start worker process 30
2022/04/23 11:41:22 [notice] 1#1: start worker process 31

kubectl logs -n prod db-0 | tail -n 6
2022-04-23 11:41:22.468 UTC [1] LOG:  starting PostgreSQL 13.6 on x86_64-pc-linux-musl, compiled by gcc (Alpine 10.3.1_git20211027) 10.3.1 20211027, 64-bit
2022-04-23 11:41:22.469 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2022-04-23 11:41:22.469 UTC [1] LOG:  listening on IPv6 address "::", port 5432
2022-04-23 11:41:22.479 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2022-04-23 11:41:22.485 UTC [20] LOG:  database system was shut down at 2022-04-23 11:41:17 UTC
2022-04-23 11:41:22.493 UTC [1] LOG:  database system is ready to accept connections
```

6. При необходимости можно сделать форвард портов и проверить работоспособность api и интерфейс фронта.
```bash
kubectl port-forward -n prod back-8557cb55c9-qfzqt  27000:9000
Forwarding from 127.0.0.1:27000 -> 9000
Forwarding from [::1]:27000 -> 9000
Handling connection for 27000
```
Если все корректно, curl http://localhost:27000/api/news/ отдаст список новостей.

Аналогичным образом фронт отдаст только разметку и заголовок. Чтобы заработало нужно делать ингресс на бэк, образ фронта собрать с указанием переменной на этот адрес.

---
<details>

  <summary>Описание задачи</summary>  
## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

</details>

### Решение


---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

В качестве решения прикрепите к ДЗ конфиг файлы для деплоя. Прикрепите скриншоты вывода команды kubectl со списком запущенных объектов каждого типа (pods, deployments, statefulset, service) или скриншот из самого Kubernetes, что сервисы подняты и работают.

---
