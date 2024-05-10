# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки с помощью Docker Compose

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск в Minikube.

Чтобы развернуть сайт в Minikube, нужно установить [Virtualbox](https://www.virtualbox.org/wiki/Downloads), [Minikube](https://minikube.sigs.k8s.io/docs/start/) и [Kubectl](https://kubernetes.io/docs/tasks/tools/). Для запуска базы данных  в кластере понадобится [Helm](https://helm.sh/). 

Перейдите в папку kubernetes и создайте кластер:
```
minikube start --vm-driver=virtualbox
```

Если при использовании Minikube в связке с Virtualbox возникают проблемы, можно использовать другой драйвер - например, [kvm2](https://minikube.sigs.k8s.io/docs/drivers/kvm2/).

В папке kubernetes создайте файл django-secrets.yaml для хранения описанных выше переменных окружения следующего вида:
```
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  ALLOWED_HOSTS: Kg==
  DEBUG: RmFsc2U=
  SECRET_KEY: cmVwbGFjZQ==
  DATABASE_URL: cG9zdGdyZXM6Ly90ZXN0X2s4czpPd090QmVwOUZydXRAcHNxbC1kYi1wb3N0Z3Jlc3FsOjU0MzIvdGVzdF9rOHM=
```
Значения переменных окружения указаны в кодировке base64.

Далее загрузите переменные окружения:
```
kubectl apply -f django-secrets.yaml
```
Создайте Docker-образ:
```
docker build -t <dockerhub_name>/django-app:latest .
```
И загрузите его в хранилище:
```
docker push <dockerhub_name>/django_app:latest
```
Подготовьте базу данных PostgreSQL:
```
helm install my-release oci://registry-1.docker.io/bitnamicharts/postgresql
```
Далее запускаем базу данных:
```
kubectl run psql-db-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.2.0-debian-12-r15 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host psql-db-postgresql -U postgres -d postgres -p 5432
``` 
Через интерфейс psql создаем таблицу, пользователя и задаем ему соответствующие разрешения.

Запустите проект Django:
```
kubectl apply -f django-deploy.yaml
```
Примените миграции:
```
kubectl apply -f django-migrate.yaml
```
Добавьте [ingress](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/):
```
kubectl apply -f ingress.yaml
```
Добавьте адрес star-burger.test в файл hosts для удоства тестирования:
```
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```
Подключите автоматический запуск очистки сессий Django:
```
kubectl apply -f django-clearsessions.yaml
```
## Запуск в Yandex Cloud.

Установите интерфейс командной строки Yandex Cloud ([CLI](https://yandex.cloud/en/docs/cli/quickstart)) и и подключитесь к кластеру. Для работы с кластером удобно использовать графический интерфейс [Lens](https://store.k8slens.dev/products/lens-desktop-personal).

Соберите образ из директории с Dockerfile:
```
docker build -t <image-name> .
```
В качестве тега образа для dev-окружения удобно использовать хэш коммита. Добавляем его при отправке в Docker Hub:
```
commit=$(git rev-parse --short HEAD) 
docker push <your-docker-id>/<image-name>:$commit
```
Манифесты для деплоя в Yandex Cloud находятся в директории deploy/yc-sirius/edu-goofy-allen. Создайте в ней файл django-secrets.yaml для хранения переменных окружения по аналогии с Minikube.

Для подключения к базе данных PostgreSQL, которая развернута в Yandex Cloud, нужен [SSL-сертификат](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect#get-ssl-cert):
```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0600 ~/.postgresql/root.crt
```
Создайте Secret:
```
kubectl create secret generic postgresql-ssl -n <namespace>  --from-file=/path_to_file/root.crt
```
### Запуск Django-приложения

В файле django-deploy.yaml измените данные образа на свои, а также укажите nodePort согласно предоставленным настройкам ALB-роутера. После этого выполните:

```
kubectl -n <namespace> apply -f django-service.yaml
```
Выполните миграции:
```
kubectl -n <namespace> apply -f django-migrate.yaml
```
Ссылка на сайт: [edu-goofy-allen.sirius-k8s.dvmn.org](https://edu-goofy-allen.sirius-k8s.dvmn.org/)

Серверная инфраструктура описана [здесь](https://sirius-env-registry.website.yandexcloud.net/edu-goofy-allen.html).
