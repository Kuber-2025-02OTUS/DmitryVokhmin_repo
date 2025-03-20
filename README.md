# Репозиторий для выполнения домашних заданий курса "Инфраструктурная платформа на основе Kubernetes-2025-02" 


## Homework 2: Знакомство с Kubernetes, основные понятия и архитектура
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
 

## Homework 4: Сетевая подсистема и сущности Kubernetes

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