# Репозиторий для выполнения домашних заданий курса "Инфраструктурная платформа на основе Kubernetes-2025-02" 

### Homework 1
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
 

