# Облако белогривых лошадок

## Авторы и вклад

1. Аладина Екатерина [@kotyablinov](https://github.com/kotyablinov):
   1. Развертывание Traefik
   2. Создание и конфигурирование приложений (`app[1...3]`) и их маршрутов до Traefik.
2. Племяшова Лидия [@Plemyashova](https://github.com/Plemyashova)
   1. Развертывание Authentik
   2. Создание и настройка приложений, управление пользователями в Authentik.

## 0. Предварительные условия

### 0.1. Инфраструктура

1. Наличие публичного домена (для демонстрации используется `whitemanedhorses.ru`).
2. Наличие VDS (для демонстрации была приобретена VDS на reg.ru).
3. Использование дистрибутивов Linux c docker и docker compose (для демонстрации используется Ubuntu 24.04 LTS).
4. Наличие `A` записей у регистраций домена на созданную на публичный адрес VDS:

   - `traefik.whitemanedhorses.ru` -> публичный IP VDS
   - `auth.whitemanedhorses.ru` -> публичный IP VDS
   - `app1.whitemanedhorses.ru` -> публичный IP VDS
   - `app2.whitemanedhorses.ru` -> публичный IP VDS
   - `app3.whitemanedhorses.ru` -> публичный IP VDS

### 0.2. Описание

- `traefik.whitemanedhorses.ru` - панель управления Traefik с авторизацией Basic Auth (по паролю из файла).
- `auth.whitemanedhorses.ru` - единая точка входа Authentik.
- `app1.whitemanedhorses.ru` - [сервис whoami](https://github.com/traefik/whoami) (сервер Go, который выводит информацию об операционной системе и HTTP-запрос на вывод) без авторизации для демонстрации динамической маршутизации Traefik.
- `app2.whitemanedhorses.ru` - сервис с авторизацией с доступом у всех пользователей.
- `app3.whitemanedhorses.ru` - сервис с авторизацией с доступом только у определенной группы пользователей.

## 1. Развертывание Traefik

Команды данного раздела нужно выполнять в папке [traefik](./traefik).

### 1.1. Создать общую сеть

В корне папки выполнить команду создания сети:

```bash
docker network create whitemanedhorses
```

Данная сеть необходима для динамического обнаружения маршрутов из docker-контейнеров.

### 1.2. Генерация паролей для панели управления с помощью htpasswd

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

### 1.3. Поднятие контейнеров

Поднять docker compose из папки `traefik`:

```bash
docker compose pull
docker compose up -d
```

### 1.4. Проверка работы Traefik

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

## 2. Развертывание Authentik

Команды данного раздела нужно выполнять в папке [authentik](./authentik).

### 2.1. Генерация секретов

Необходимо [сгенерировать секреты](https://docs.goauthentik.io/docs/install-config/install/docker-compose). Находясь в папке `authentik` выполнить команду:

```bash
echo "COMPOSE_PROJECT_NAME=whitemanedhorses-authentik" >> .env
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
```

### 2.2. Поднятие контейнеров

Поднять docker compose из папки `authentik`:

```bash
docker compose pull
docker compose up -d
```

После поднятия контейнеров в маршрутах Traefik отобразится новый до Authentik:

![](images/traefik-dashboard-routers-2.png)

### 2.3. Первоначальная настройка

Чтобы начать первоначальную настройку, перейдите по ссылке `auth.whitemanedhorses.ru/if/flow/initial-setup/`. Вести почту и логин от аккаунта `akadmin`. Этот пользователь будет администратором по умолчанию.

### 2.4. Проверка работы Authentik

После входа по логину и паролю будет доступен панель управления пользователя:

![](images/authentik-user-dashboard-1.png)

Панель управления администратора:

![](images/authentik-admin-dashboard-1.png)

## 3. Создание новго приложения app2 в Authentik

Действия далее выполняются в панели управления администратора Authentik.

### 3.1. Создание нового приложения

Перейти в интерфейс администратора. В вкладке приложения открыть страницу "Приложения". Вызвать окно создания приложения, нажав кнопку "Create with Wizard".

Указать имя `app2`, идентификатор `app2` и URL запуска:

![](images/authentik-app2-1.png)

На следующей вкладке (нажать далее), выбрать Proxy Provider:

![](images/authentik-app2-2.png)

В настройках прокси провайдера выбрать поток авторизации `default-provider-authorization-explicit-consent`, переадресацию аутентификациии (одно приложение) и указать внешний хост:

![](images/authentik-app2-3.png)

На остальных вкладках оставить значения по умолчанию. На последней вкладке выбрать "отправить".

### 3.2. Создание внешнего компонента

Перейти в разделе "Приложения" на страницу "Внешние компоненты". Создать компонент, нажав на кнопку "создать".

На форме создания задать необходимые параметры и выбрать приложение `app2`:

![](images/authentik-app2-4.png)

### 3.3. Копирование токена AUTHENTIK_TOKEN

Для работы Forward Auth необходимо скопировать значение `AUTHENTIK_TOKEN`. Это можно сделать через действия внешнего компонента:

![](images/authentik-app2-5.png)

## 4. Развертывание app2

Команды данного раздела нужно выполнять в папке [app2](./app2).

### 4.1. Генерация секретов

Находясь в папке `app2` выполнить команду:

```bash
echo "COMPOSE_PROJECT_NAME=whitemanedhorses-app2" >> .env
echo "AUTHENTIK_TOKEN=<ТОКЕН ДЛЯ ВНЕШНЕГО КОМПОНЕНТА, ПОЛУЧЕННЫЙ РАНЕЕ ДЛЯ APP2>" >> .env
```

### 4.2. Поднятие контейнеров

Поднять docker compose из папки `app2`:

```bash
docker compose pull
docker compose up -d
```

### 4.3. Проверка приложения app2

Открыть в браузере `app2.whitemanedhorses.ru`. Будет открыта страничка с данными для входа. Если ранее был осуществлен вход в Authentik, то появится следующее уведомление:

![](./images/app2-login-1.png)

Если принять соглашения, то произойдет редирект в приложение `app2`:

![](./images/app2-login-2.png)

## 5. Создание нового пользователя

Действия далее выполняются в панели управления администратора Authentik.

Для демонстрации различных прав доступа, создадим еще одного пользователя.

В интерфейсе администратора в вкладке "Каталог" перейти на страницу "Пользователи". Открыть форму создания пользователя, нажав на кнопку "Создать". Указать данные для нового пользователя:

![](images/create-user-1.png)

Перейти на страницу созданного пользователя и задать пароль:

![](images/create-user-2.png)

## 6. Создание нового приложения app3 в Authentik

Действия далее выполняются в панели управления администратора Authentik.

Для начала, требуется повторить все те же действия, что и в разделе 3, только вместо `app2` указывать `app3`.

Далее открыть приложение `app3` со страницы приложений:

![](images/app3-groups-1.png)

Перейти на вкладку "Политика/Пользователь/Пользовательские привязки".

Нажать на кнопку "Создать и привязать политику". Задать политику, которая ограничит доступ к `app3` только для администраторов Authentik:

![](images/app3-groups-2.png)

В результате:

- У администратора по умолчанию будет доступ к приложению `app3`.
- У пользователя, созданного в разделе 5, доступа к приложению не будет.

## 7. Развертывание app3

### 7.1. Генерация секретов

Находясь в папке `app3` выполнить команду:

```bash
echo "COMPOSE_PROJECT_NAME=whitemanedhorses-app3" >> .env
echo "AUTHENTIK_TOKEN=<ТОКЕН ДЛЯ ВНЕШНЕГО КОМПОНЕНТА, ПОЛУЧЕННЫЙ РАНЕЕ ДЛЯ APP3>" >> .env
```

### 7.2. Поднятие контейнеров

Поднять docker compose из папки `app3`:

```bash
docker compose up -d
```

### 7.3. Проверка приложения app3

Открыть в браузере `app3.whitemanedhorses.ru`. Будет открыта страничка с данными для входа. Если ранее был осуществлен вход в Authentik администратором по умолчанию, то появится следующее уведомление:

![](images/app3-login-1.png)

Если принять соглашения, то произойдет редирект в приложение `app3`. Страница будет аналогична приложению `app2`.

Если был осуществлен вход в Authentik обычным пользователем, то будет отказано в доступе:

![](images/app3-login-2.png)

## 8. Итоговые маршруты Traefik

В результате проделанных выше действий получим следующие HTTP-маршруты Traefik:

![](images/traefik-dashboard-routers-3.png)

## Выводы

Успешно выполнена настройка Traefik и Authentik, а также развертывание приложений с различными уровнями доступа.

В соответствии с требованиями:

1. Существует Auth (в виде Authentik):
   1. Хранится отображение вида User: [Available Resources] - реализовано в Authentik на вкладке Пользователи.
   2. Отображение можно получить и обновить по API для каждого пользователя - для каждого пользователя можно редактировать доступ к приложению в Authentik.
2. Существует Router (в виде Traefik):
   1. Выполняет свою базовую функцию - перенаправляет запросы пользователей - Traefik перенаправляет запросы пользователей.
   2. Имеет API для добавления или удаления правил перенаправления - Traefik поддерживает динамическое изменение перенаправлений.
   3. Умеет проверять права пользователей на доступ к необходимому ресурсу - данная задача решается в Authentik.
3. Создано 3 приложения с одним и тем же функционалом, но разными правами доступа для различных пользователей:
   1. К `app1` можно подключиться даже неавторизированным пользователям.
   2. К `app2` могут подключиться только авторизированные пользователи.
   3. К `app3` подключаются только пользователи с правами администратора.

## Источники

1. [Authentik: Single Sign-On for Your Self-Hosted Apps (Forward Auth and OAuth2)](https://www.youtube.com/watch?v=ywQVe9ikcVI) - базовые концепции SSO в Authentik.
2. [Документация Traefik](https://doc.traefik.io/traefik/).
3. [Simple HTTPs for Docker! // Traefik Tutorial](https://www.youtube.com/watch?v=-hfejNXqOzA) - краткий гайд по Let's Encrypt.
4. [Setup SSL with Traefik and Let's Encrypt](https://thomasventurini.com/articles/setup-ssl-with-traefik-and-lets-encrypt/) - еще один краткий гайд по Let's Encrypt.
5. [Let's Encrupt документация в Traefik](https://doc.traefik.io/traefik/user-guides/docker-compose/acme-tls/)
6. [Документация Authentik](https://docs.goauthentik.io/docs/).
7. [Forward authentication Traefik | Authentik](https://docs.goauthentik.io/docs/add-secure-apps/providers/proxy/server_traefik) - проксирование Forward Auth в Traefik.
8. [Secure authentication for EVERYTHING! // Authentik](https://www.youtube.com/watch?v=N5unsATNpJk) - краткий гайд по Authentik.