---
title: "Управление сетями и портами"
description: "Как создавать пользовательские сети, связывать контейнеры и пробрасывать порты."
---


## Проброс портов (Port Publishing)

Чтобы сделать сервис доступным извне (с хоста или интернета), нужно опубликовать порты.

```bash
# Синтаксис: -p <HOST_PORT>:<CONTAINER_PORT>

# Открыть порт 80 контейнера на порту 8080 хоста
docker run -p 8080:80 nginx

# Открыть только на localhost (безопасно для баз данных)
docker run -p 127.0.0.1:5432:5432 postgres
```

## Работа с пользовательскими сетями

Создание собственной сети — лучшая практика для связи контейнеров.

### 1. Создание сети
```bash
docker network create my-app-net
```

### 2. Запуск контейнеров в сети
```bash
# Запускаем базу данных (имя контейнера = DNS имя)
docker run -d --name db --network my-app-net postgres

# Запускаем приложение, которое использует БД
docker run -d --name web --network my-app-net -p 80:80 my-web-app
```

Теперь `web` может обращаться к `db` просто по хостнейму `db`.

### 3. Подключение работающего контейнера
Если контейнер уже запущен, его можно подключить к новой сети на лету:
```bash
docker network connect my-app-net existing-container
```

## Диагностика

### Просмотр сетей
```bash
docker network ls
```

### Детали сети
Чтобы узнать, какие контейнеры подключены к сети и их IP-адреса:
```bash
docker network inspect my-app-net
```

### Проверка связи
Используйте временный контейнер `alpine` или `busybox` для проверки DNS и пинга:
```bash
docker run --rm --network my-app-net alpine ping db
# Должен ответить IP адрес контейнера db
```

## Связанные материалы

- [[3-networking-modes|Теория сетевых режимов]]
- [[devops/docker/tutorials/docker-compose-basics|Сети в Docker Compose (проще!)]]
