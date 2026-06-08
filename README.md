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

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Настройка Kubernetes Secret

Перед деплоем необходимо создать Secret с конфиденциальными данными.

1. Создайте файл `kubernetes/secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  SECRET_KEY: "ваш-секретный-ключ"
  DATABASE_URL: "postgres://user:password@host:port/dbname"
```

2. Примените Secret
```bash
kubectl apply -f kubernetes/secret.yaml
```   

3. Проверка 
```bash
kubectl get secret django-secret
kubectl describe secret django-secret
``` 

## Доступ к сайту

Сайт доступен по доменному имени `star-burger.test`.

### Настройка локального DNS

Добавить запись в файл `hosts`:

**Windows:** 
`192.168.49.2 star-burger.test`

Файл находится по пути: `C:\Windows\System32\drivers\etc\hosts`

**Linux/Mac:**
```bash
echo "$(minikube ip) star-burger.test" | sudo tee -a /etc/hosts
```

### Запуск сайта

1. Загрузите образ Django в Minikube:
```bash
minikube image load django_app:latest
```

2. Запустите базу данных (если не запущена):
```bash
docker start pg-external
```

3. Примените манифесты:
```bash
kubectl apply -f kubernetes/secret.yaml
kubectl apply -f kubernetes/django-deployment.yaml
kubectl apply -f kubernetes/django-service.yaml
```

4. Получить URL:
```bash
minikube service django --url
```

5. Откройте сайт в браузере по полученному URL или по адресу http://star-burger.test

# Проверка

## Узнать IP Minikube
```bash
minikube ip
```

## Проверить доступность домена
```bash
ping star-burger.test
```

# Автоматическая очистка сессий (CronJob)

Для автоматической очистки устаревших сессий настроен CronJob, который запускает команду `clearsessions` каждый день в 6:00 утра.

### Манифест

Файл: `kubernetes/django-cronjob.yaml`

### Проверка

```bash
# Посмотреть CronJob
kubectl get cronjobs

# Посмотреть Jobs
kubectl get jobs

# Принудительно запустить очистку сессий
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-once

# Проверить статус Pod
kubectl get pods | grep clearsessions
```

### Открыть сайт
http://star-burger.test