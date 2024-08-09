# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Оглавление
- [Как подготовить окружение к локальной разработке](#как-подготовить-окружение-к-локальной-разработке)
  - [Как запустить сайт для локальной разработки](#как-запустить-сайт-для-локальной-разработки)
  - [Как вести разработку](#как-вести-разработку)
  - [Переменные окружения](#переменные-окружения)
  - [Создание кластера](#создание-кластера)
  - [Создание БД внутри кластера](#создание-бд-внутри-кластера)
  - [Создание Secrets](#создание-secrets)
  - [Создание Ingress](#создание-ingress)
  - [Запуск CronJob](#запуск-cronjob)
  - [Запуск Job](#запуск-job)
- [Разработка в Yandex Cloud](#разработка-в-yandex-cloud)

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

## Создание кластера
Скачайте [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download).
Для запуска кластера выполните:
```shell
minikube start --driver=virtualbox
```
Minikube запускает свой собственный экземпляр Docker демона внутри виртуальной машины. Чтобы управлять контейнерами, созданными внутри этой VM, с вашей локальной машины через docker команды, необходимо перенастроить Docker клиент на использование демона Minikube. 
Для windows выполните:
```shell
minikube docker-env --shell powershell | Invoke-Expression
```
Для linux выполните:
```shell
eval $(minikube docker-env)
```
Далее, соберите образ и загрузите его в Minikube
```shell
docker compose build
minikube image build . -t django_app
```
## Создание БД внутри кластера
Для создания БД внутри кластера использовался [Helm](https://helm.sh/).

После установки Helm для скачивания пакета Postgresql и подключения к БД выполните:
```shell
helm install your-pod-name --set auth.username=your-name --set auth.password=your-password --set auth.database=your-database-name oci://registry-1.docker.io/bitnamicharts/postgresql 
```
`DATABASE_URL` для запуска БД должна иметь значение `postgres://[your-name]:[your-password]@[your-pod-name]-postgresql:5432/[your-database-name]` 

## Создание Secrets

Для создания Secrets файла необходимо создать YAML-файл следующего содержания:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
data:
  ALLOWED_HOSTS: your_allowed_hosts_base64_encode
  DATABASE_URL: your_db_url_base64_encode
  SECRET_KEY: your_secret_key_base64_encode
  DEBUG: your_debug_base64_encode
```
Для кодирования данных в base64 в Windows воспользуйтесь командой powershell:
```powershell
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes(‘text_to_encode’))
```
Для кодирования данных в base64 в Linux воспользуйтесь командой powershell:
```shell
echo -n 'text_to_encode' | base64```
```
Затем, в кластере необходимо создать Secrets файл:
```shell
kubectl apply -f path/to/your/secrets.yaml
```
## Создание Ingress
Создадим YAML-файл для Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your_ingress_name
spec:
  ingressClassName: nginx
  rules:
    - host: your_host
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your_service_name
                port:
                  number: your-port
```
Для создания Ingress запускаем:

```shell
minikube addons enable ingress
kubectl apply -f path/to/your/ingress.yaml
kubectl apply -f path/to/deployment_service.yaml
```

Добавьте данный текст в файл hosts:

```
[вывод команды minikube ip] your-host-name
```

После этого сайт будет доступен по адресу, который вы указали.

## Запуск CronJob
Для ежемесячной очистки сессий Django, воспользуйтесь командой:
```shell
kubectl apply -f path/to/cronjob-clearsessions.yaml
```
Для запуска в любой момент времени Job из CronJob выполните команду:
```shell
kubectl create job --from=cronjob/<cronjob-name> <job-name> -n <namespace-name>
```

## Запуск Job
Для запуска команды migrate в контейнере выполните:
```shell
kubectl apply -f path/to/job-migrate.yaml
```
Для проверки вывода команды migrate выполните:
```shell
kubectl get pods
kubectl logs migrate-pod-name
```

## Разработка в Yandex Cloud
В качестве UI для терминала можно использовать [k9s](https://k9scli.io/), в качестве UI для k8s [Lens Desktop](https://k8slens.dev/)
Для запуска сервиса в yandex cloud необходимо:
1. Подключиться к кластеру Yandex cloud
   1. [Установите и инициализируйте интерфейс командной строки Yandex Cloud](https://yandex.cloud/ru/docs/cli/quickstart#install)
   2. [Добавьте учетные данные](https://yandex.cloud/ru/docs/managed-kubernetes/operations/connect#kubectl-connect) кластера Kubernetes в конфигурационный файл kubectl:
     ```
     yc managed-kubernetes cluster get-credentials --id catvr0kq2b7bn0v8k2r9 --external
     ```
2. Используйте утилиту kubectl для работы с кластером Kubernetes:
    ```
      kubectl get cluster-info
      kubectl get pods --namespace=<your-namespace>
    ```

### Запуск Service и Deployment из манифеста
После подключения к кластеру, запустите создание Service и Deployment из манифеста:
```
kubectl create --namespace=<your-namespace> -f ./path/to/deployment_service.yaml
```
Теперь nginx доступен по адресу хоста вашего ALB, в моем случае https://edu-evil-panini.sirius-k8s.dvmn.org/
