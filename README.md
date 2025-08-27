# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его
установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции
будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker-compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web python /webapp/src/manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web python /webapp/src/manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по
адресу [http://127.0.0.1:8080/admin/](http://127.0.0.1:8080/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не
требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции
схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие
миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой
библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно
лишь, чтобы оно никому не было
известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или
`FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт
ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например
`127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не
поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск сайта через K&S minikube

Перейдите в основной каталог проекта, где находится файл `docker-compose.yml`

В этой директории необходимо создать файл `.env` и заполнить его **обязательными** переменными окружения:

- SECRET_KEY
- DATABASE_URL

После заполнения переменных окружения перейдите в каталог `backend_main_django`.

Создайте образ в minikube, запустив команду:

```bash
minikube image build . -t django-app:latest
```

Далее, перейдите в каталог `k&s` (находится каталогом выше) и введите команду, которая создает Secret:

```bash
kubectl create secret generic django-secret --from-env-file=.env
```

Для создания и запуска деплоймента, введите команду:

```bash
kubectl apply -f deployment.yaml
```

Для запуска сервиса, введите команду:

```bash
kubectl apply -f service.yaml   
```

Для запуска ingress, введите команду:

```bash
kubectl apply -f ingress.yaml   
```

Для доступности сайта через браузер необходимо добавить в файл [
`hosts`](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit)
запись

```bash
<minicube_ip> star-burger.test
```

## Запуск базы данных

Установите [helm](https://helm.sh/)

Введите команду, заполнив данные БД:

```bash
helm uninstall my-postgres; helm install my-postgres oci://registry-1.docker.io/bitnamicharts/postgresql --set auth.database=<DATABASE> --set auth.username=<USERNAME> --set auth.password=<PASSWORD> --set primary.service.type=NodePort
```

## Management команды в k&s

Перед выполнением команд необходимо быть в каталоге `k&s` (находится внутри проекта).

### Регулярное удаление сессий.

Запускается командой:

```bash
kubectl apply -f clearsessions-cronjob.yaml
```

По умолчанию, сессии удаляются каждую минуту. При необходимости можно значение изменить в файле
`clearsissions-cronjob.yaml`.

### Migrate

Запускается командой:

```bash
kubectl apply -f migrate-job.yaml
```

