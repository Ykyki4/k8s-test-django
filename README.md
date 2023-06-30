# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Как запустить проект в kubernetes

Для начала, узнайте свой внешний айпи адрес. [Windows](https://support.microsoft.com/en-us/windows/find-your-ip-address-in-windows-f21a9bbc-c582-55cd-35e0-73431160a1b9) [Linux](https://www.ionos.com/digitalguide/hosting/technical-matters/get-linux-ip-address/#:~:text=If%20you%20enter%20the%20command,that%20are%20in%20your%20network.) [MacOs](https://www.wikihow.com/Find-Your-IP-Address-on-a-Mac), далее в одном каталоге с [`docker-compose.yml`](./docker-compose.yml) создайте файл `docker-compose.override.yaml` с данным содержимым:

```yaml
version: "3"

services:
  db:
    ports:
      - [ваш внешний айпи адрес]:[предпочитаемый порт]:5432
```

После этого запустит базу данных командой:

```shell-session
docker-compose up db
```

Следуя данному [видео](https://www.youtube.com/watch?v=q_nj340pkQo&list=PLg5SS_4L6LYvN1RqaVesof8KAf-02fJSi&index=1) запустите кластер, загрузите в кластер образ джанго данной командой:

```shell-session
minikube image build -t django_app:v1.0 ./backend_main_django/Dockerfile
```

Далее создайте файл с конфигурацей в формате .yaml для запуска сайта:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config-v1
  labels:
    app.kubernetes.io/name: django
    app.kubernetes.io/instance: django-config
    app.kubernetes.io/version: "1.0"
data:
  ALLOWED_HOSTS: star-burger.test
  SECRET_KEY: Описание переменной находится ниже
  DATABASE_URL: postgres://test_k8s:OwOtBep9Frut@айпи и порт на которых запущена база данных/test_k8s
  DEBUG: Описание переменной находится ниже, важно, чтобы переменная находилась в одинарных ковычках, вроде: 'False'
```

Запускаем ряд команд:

```shell-session
kubernetes apply -f путь к только что созданному конфигурационному файлу.
kubernetes apply -f ./kubernetes/deployments/django-deployment.yaml
kubernetes apply -f ./kubernetes/services/django-service.yaml
kubernetes apply -f ./kubernetes/ingress/django-ingress.yaml
```

Добавьте данный текст в etc/hosts:

```
[вывод команды minikube ip] star-burger.test
```

После этого сайт будет доступен по адрессу star-burge.test

Также, вы можете добавить автоматическое удаление истекших джанго сессий:
```
kubernetes apply -f ./kubernetes/cronjobs/django-clearsessions.yaml
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
