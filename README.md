# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

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

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить проект в minikube

[Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) и [minikube](https://minikube.sigs.k8s.io/docs/) должны быть установлены и настроены.


Создавем файл конфигурации `configmap.yaml`:

```yaml
data:
  SECRET_KEY: <секретный ключ Django>
  DEBUG: <"False" или "True"> 
  ALLOWED_HOSTS: star-burger.test
  DATABASE_URL: <адрес для подключения к базе данных PostgreSQL. 
```

Загружаем configmap командой
```sh
kubectl apply -f deploy/local-minikube-virtualbox/django-app-configmap.yaml
```

Запустите deployment командой
```sh
kubectl apply -f deploy/local-minikube-virtualbox/django-app-deployments.yaml
```

Включите Ingress:
```sh
minikube addons enable ingress
```

После этого необходимо применить настройки Ingress:
```sh
kubectl apply -f deploy/local-minikube-virtualbox/django-app-ingress.yaml
```

Примените миграции Django к базе данных:
```sh
kubectl apply -f deploy/local-minikube-virtualbox/django-migrate.yaml
```

Создайте запланированную задачу, удаления сессий Django:
```sh
kubectl apply -f deploy/local-minikube-virtualbox/django-clearsessions.yaml
```

## Как запустить в yandex-sirius облаке

### Создание Docker образа
Вы можете использовать готовый Docker образ, который залит на DockerHub. Так же есть возможность создать свой собственный.
Создайте репозиторий на [DockerHub](https://hub.docker.com/)
Для этого перейдите в папку `backend_main_django` и соберите образ

```sh
docker build  -t yournaname/appname:tagname . 
```

Сделайте `push`
```sh
docker login
docker push yournaname/appname:tagname
```

### Запуск в yandex-sirius облаке

Задайте `context` и `namespace`
```sh
kubectl config use-context yc-sirius
kubectl config set-context --current --namespace=edu-goofy-allen
```

Cкачайте сертификат 
```sh
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem"
```

Создайте секрет
```sh
kubectl create secret generic psql-cert --from-file=root.crt
```

Перейдите в папку `yc-sisrius/edu-admiring-carson` и примените конфигурационные файлы
```sh
kubectl apply -f django-app-config.yaml
kubectl apply -f django-secret.yaml
kubectl apply -f django-app-deployments.yaml
kubectl apply -f django-app-ingress.yaml 
```

Примените миграции Django к базе данных:
```sh
kubectl apply -f django-migrate.yaml 
```

Создайте запланированную задачу, удаления сессий Django:
```sh
kubectl apply -f django-clearsessions.yaml
```

Создание  superuser
При первом запуске так же необходимо создать superuser
Перейдите в нужный контейнер `django-app`

```sh
kubectl exec -it django-app-deployment  -c django-app -- /bin/sh
```

создайте superuser
```sh
python manage.py createsuperuser
```

[Пример сайта](https://edu-admiring-carson.sirius-k8s.dvmn.org/admin/login/?next=/admin/)