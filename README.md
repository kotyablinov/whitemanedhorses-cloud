# whitemanedhorses-cloud

Для демонстрации используется Ubuntu 24.04 LTS.

## Развертывание Traefik

Для начала нужно перейти в папку [traefik](./traefik).

### Шаг 1. Создать общую сеть

В корне папки выполнить команду создания сети:

```bash
docker network create whitemanedhorses
```

### Шаг 2. Генерация паролей для дашборды с помощью htpasswd

`htpasswd` - Управление пользовательскими файлами для базовой аутентификации. Он требуется для обеспечения входа в панель управления Traefik по паролю. Для установки выполнить команды: 

```
apt update
apt install apache2-utils -y
```

Сгенерировать файл с паролем:

```bash
echo $(htpasswd -nB user) | sed -e s/\\$/\\$\\$/g > ./users/passwords
```
