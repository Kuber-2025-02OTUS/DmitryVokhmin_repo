# Репозиторий для выполнения домашних заданий курса "Инфраструктурная платформа на основе Kubernetes-2025-02" 


## Homework 1: Знакомство с Kubernetes, основные понятия и архитектура
 Папка ./kubernetes-intro
 в папке 3 файла:
 - namespace.yaml - создается namespace homework
 - configmap.yaml - создается configmap c настройкой nginx, для раздачи файлов с директории /homework на 8000 порту
 - pod.yaml - создается pod с подключением к configmap и подключением в namespace homework
    В pod-е создается init-container, который скачивает в папку /homework .zip архив, распаковывает и удаляет его.
    После этого работа init-container завершается.
    Далее запускается основной контейнер, nginx который посредством configmap.yaml настроен на раздачу файлов из папки /homework, а именно файла index.html в папке /homework/ready-html . Если запустить можно увидеть крассивую веб-страницу.

    В секции preStop.exec.command описано удаление всех файлов из папки /homework. Поскольку я разворачивал не отдельный файл, а целую папку, то при удалении папки, удаляются все файлы.

Для развертывания файлы накатываются последовательно через kubectl apply -f <файл yaml> в следующем порядке:
 - namespace.yaml
 - configmap.yaml
 - pod.yaml

## Homework 2: Kubernetes controllers. ReplicaSet, Deployment,
1. Cоздан манифест namespace.yaml для namespace с именем homework.
(*На самом деле на прошлом занятии создан, но прилагаю файл к этому 
домашнему заданию также.*)

2. Создан манифест Deployment.
  - Указанн namespace `homework`
  - Заданно количество реплик `replicas: 3`
  - Добавлена `readnessProbe`:
```yaml
  readinessProbe:
    exec:
      command:
      - test
      - -f
      - /homework/ready-html/index.html
    initialDelaySeconds: 5
    periodSeconds: 5
```
  - добавлена стратегия обновления:
  ```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  ```

3. (Задание*) Добавить условие запуска подов на нодах с меткой.
  - добавлена метка к ноде (для миникуба по своему задается)
```shell
kubectl label nodes minikube homework=true
```
  - добавлено условие в deployment
```yaml
  nodeSelector:
    homework: "true"
```

## Homework 3: Сетевая подсистема и сущности Kubernetes

Папка ./kubernetes-network
Домашняя работа - продолжение работы проделанной в homework 2.
Используется тот же namespace, тот же Config Map. Тот же Pod  - но с небольшими изменениями.

В проекте оставлены без изменения:
- configmap.yaml

В проекте изменены файлы:
- pod.yaml

В проект добавлены новые файлы:
- service.yaml
- ingress.yaml
- clusterissuer.yaml

В pod.yaml добавлены labels для увязки с сервисом

```yaml
  labels:
    app: homework-site-pod
```

service.yaml - заводит трафик с объявленного порта 8000 на порт контейнера.
Обратите внимание что порт пода адресуется по имени, а не по номеру, что позволяет определять номер порта только в одном месте.

- ingress.yaml - заводит http/https трафик c портов 80/443 на сервис с именем homework-service.
B ingress добавлено rewrite правило которое переписывает url запрос таким образом что все запросы на '/homepage'  переводит в '/' таким образом выполняется задание со звездочкой.

Были добавлены аннотации для rewrite и регулярных выражений
```yaml
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
```
регулярное выражение в path 
```yaml
- path: /homepage(/.*|$)|/(.*)
```

В рамках домашнего задания был активрован ingress контроллер nginx в minikube который был там установлен но деактивирован.


- clusterissuer.yaml - создает сертификат для ingress, позволяя ему работать по протоколу https.
 Для работы по протоколу https В ingress.yaml были добавлены следующие секции.

Секция для генерации сертификата и сохранения секрета в kubernetes
```yaml
  tls:
  - hosts:
    - homework.otus
    secretName: homework-otus-tls
```

Аннотация для использования сертификата и редиректа с http на https
```yaml
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Для NGINX Ingress
```

## Homework 4: Volumes, StorageClass, PV, PVC
Папка ./kubernetes-volumes

В рамках домашнего задания было сделано следующее.
- создан манифест pvc.yaml запрашивающий хранилие с storageClass по-умоланию.
Для того чтобы использовался storageClass по-умоланию cледует убрать описание `  storageClassName: ...`

- был создан (ранее) манифест config map: configMap.yaml который заводит в под файл с конфигурацией nginx в папку: /etc/nginx/conf.d/ (немного не точно по заданию, но этот configMap здесь прям нужен чтобы сконфигурировать nginx)

- был создан манифест cm.yaml создаюший значения key-value для использования в поде, 
соответсвенно под эти значения были добавлены записи:
```yaml
volumes:
  ...
  - name: configdir
    configMap:
      name: homework-config
      items:
      - key: database_url
        path: database_url
      - key: api_key
        path: api_key
      - key: debug_mode
        path: debug_mode
```

а также монтирование каталога с файлами этих значений
```yaml
  - name:  configdir
    mountPath: /homework/conf
```


- манифест deployment.yaml поправлен для использования созданного ранее pvc.yaml
Изменён следующий фрагмент:
```yaml
      volumes:
      - name: workdir
        persistentVolumeClaim:
          claimName: homework-pvc
```

- В манифесте deployment.yaml добавлено(ранее) использование созданной configMap
```yaml
      volumes:
        ...
      - name: nginx-config-volume
        configMap:
          name: nginx-config
        ...
```

и здесь:
```yaml
        volumeMounts:
        ...
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d/
        ...
```

### Задание с *
 - Был создан storageClass: storageclass.yaml с указанным reclaimPolicy и provisioner
 - Был создан PVC для использования этого storageClass: pvc-storage-class.yaml


## Homework 5: Настройка сервисных аккаунтов и ограничение прав для них
Папка ./kubernetes-security

1. Было создано: 
 - ServiceAccount c именем `monitoring` в namespace `homework` (serviceaccount.yaml)
 - ClusterRole c именем `monitoring-metrics` (clusterrole.yaml)
 - ClusterRoleBinding c именем `monitoring-metrics-binding` (clusterrolebinding.yaml)

2. Был изменен манифест deployment.yaml из предыдущего задания, чтобы он использовал созданный ServiceAccount (deployment.yaml)
3. Было создано:
  - ServiceAccount c именем `cd` в namespace `homework` (serviceaccount.yaml)
  - Role c именем `admin` в namespace `homework` (role.yaml)
  - RoleBinding c именем `cd-admin-binding` в namespace `homework` (rolebinding.yaml)

4. Создан токен доступа для ServiceAccount `cd` в namespace `homework` (файл: token)
```yaml
kubectl create token cd -n homework --duration=24h > token
```


5. Создан kubeconfig для ServiceAccount `cd` в namespace `homework` (файл: cd-kubeconfig.yaml)
  - Cluser Name: `kubectl config view --minify -o jsonpath='{.clusters[0].name}'`
  - Cluster Server: `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'`
  - certificate-authority-data:`cat /home/vokhmin/.minikube/ca.crt | base64 | tr -d '\n'`

### Задание с *
  deployment.yaml был модифицирован чтобы при старте initContainer получать метрики с API сервера и сохранять их в /init/ready-html/metrics.html
  Для этого был изменен блок initContainers[0].command в deployment.yaml для получения метрик и сохранения их в файл /init/ready-html/metrics.html
  В основном контейнере этот файл будет доступен в директории /homework/ready-html/metrics.html
  Просмотр страницы будет доступен по адресу https://homework.otus/metrics.html