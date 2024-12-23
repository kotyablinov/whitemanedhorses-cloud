# whitemanedhorses-cloud

## 0 Предварительные условия

### Инфраструктура

1. Наличие публичного домена (для демонстрации используется `whitemanedhorses.ru`).
2. Наличие VDS (для демонстрации была приобретена VDS на reg.ru).
3. Использование дистрибутивов Linux c docker и docker compose (для демонстрации используется Ubuntu 24.04 LTS).
4. Наличие `A` записей у регистраций домена на созданную на публич347334ый адрес VDS:

   - `traefik.whitemanedhorses.ru` -> публичный IP VDS
   - `auth.whitemanedhorses.ru` -> публичный IP VDS
   - `app1.whitemanedhorses.ru` -> публичный IP VDS
   - `app2.whitemanedhorses.ru` -> публичный IP VDS
   - `app3.whitemanedhorses.ru` -> публичный IP VDS

### Описание

- `traefik.whitemanedhorses.ru` - панель управления Traefik с авторизацией Basic Auth (по паролю из файла).
- `auth.whitemanedhorses.ru` - единная точка входа Authentik.
- `app1.whitemanedhorses.ru` - [сервис whoami](https://github.com/traefik/whoami) (сервер Go, который выводит информацию об операционной системе и HTTP-запрос на вывод) без авторизации для демонстрации динамической маршутизации Traefik.
- `app2.whitemanedhorses.ru` - //TODO (сервис с авторизацией с доступом только у пользователя 1)
- `app3.whitemanedhorses.ru` - //TODO (сервис с авторизацией с доступом только у пользователя 2)

## 1 Развертывание Traefik

Команды данного раздела нужно выполнять в папке [traefik](./traefik).

### 1.1 Создать общую сеть

В корне папки выполнить команду создания сети:

```bash
docker network create whitemanedhorses
```

Данная сеть необходима для динамического обнаружения маршрутов из docker-контейнеров.

### 1.2 Генерация паролей для дашборды с помощью htpasswd

`htpasswd` - Управление пользовательскими файлами для базовой аутентификации. Он требуется для обеспечения входа в панель управления Traefik по паролю. Для установки выполнить команды:

```bash
apt update
apt install apache2-utils -y
```

Сгенерировать файл с паролем:

```bash
echo $(htpasswd -nB user) > ./users/passwords
```

Будет создан пользователь `user` с паролем, который будет введен в консоли.

### 1.3 Поднятие контейнеров

Поднять docker compose из папки `traefik`:

```bash
docker compose pull
docker compose up -d
```

### 1.4 Проверка работы Traefik

Открыть в браузере `traefik.whitemanedhorses.ru`.

Ввести логин и пароль, который был установлен на предыдущем шаге.

Откроется панель управления:

![](./images/traefik-dashboard-auth.png)

Перейти на вкладку HTTP. Откроются текущие доступные маршруты:

![](./images/traefik-dashboard-routers-1.png)

Можно увидеть маршрут для `app1.whitemanedhorses.ru`:

![](./images/traefik-dashboard-app1.png)

Открыть в браузере `app1.whitemanedhorses.ru`. Будет открыта страничка с данными из HTTP заголовка запроса (без авторизации):

![](./images/app1-whoami.png)

## 2 Развертывание Authentik

Команды данного раздела нужно выполнять в папке [authentik](./authentik).

### 2.1 Генерация секретов

Необходимо [сгенерировать секреты](https://docs.goauthentik.io/docs/install-config/install/docker-compose). Находясь в папке `authentik` выполнить команду:

```bash
echo "COMPOSE_PROJECT_NAME=whitemanedhorses-authentik" >> .env
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
```

### 2.2 Поднятие контейнеров

Поднять docker compose из папки `authentik`:

```bash
docker compose pull
docker compose up -d
```

### 2.3 Первоначальная настройка

Чтобы начать первоначальную настройку, перейдите по ссылке `auth.whitemanedhorses.ru/if/flow/initial-setup/`. Вести почту и логин от аккаунта `akadmin`. Этот пользователь будет администратором по умолчанию.

### 2.4 Проверка работы Authentik

После входа по логину и паролю будет доступен панель управления пользователя:

![](images/authentik-user-dashboard-1.png)

Панель управления администратора:

![](images/authentik-admin-dashboard-1.png)
