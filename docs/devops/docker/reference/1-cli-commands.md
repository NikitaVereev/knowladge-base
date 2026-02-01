---
title: 1 Шпаргалка по Docker CLI
type: reference
tags: [docker, cli, commands, cheat-sheet, flags]
---

# Шпаргалка по Docker CLI

Краткий справочник основных команд Docker CLI.

## Containers (Lifecycle)

Управление жизненным циклом контейнеров.

| Команда | Описание |
| :--- | :--- |
| `docker run [opts] <image>` | Создать и запустить новый контейнер. |
| `docker start <container>` | Запустить остановленный контейнер. |
| `docker stop <container>` | Остановить контейнер (SIGTERM → 10s → SIGKILL). |
| `docker kill <container>` | Принудительно убить контейнер (SIGKILL). |
| `docker restart <container>` | Перезапустить контейнер. |
| `docker pause/unpause` | Приостановить/возобновить все процессы в контейнере (SIGSTOP). |
| `docker rm <container>` | Удалить остановленный контейнер. |
| `docker rm -f <container>` | Принудительно удалить работающий контейнер. |

### Ключевые флаги `docker run`

*   `-d` (`--detach`) — Запуск в фоновом режиме.
*   `--rm` — Удалить контейнер автоматически после остановки.
*   `-p host:container` — Публикация портов (например, `-p 8080:80`).
*   `-v host:container` — Bind Mount тома или директории.
*   `--name <name>` — Задать имя контейнеру.
*   `-e KEY=VAL` — Передать переменную окружения.
*   `--network <net>` — Подключить к сети.
*   `--restart <policy>` — Политика рестарта (`always`, `on-failure`, `unless-stopped`).
*   `--user <uid:gid>` — Запуск от имени пользователя.

## Containers (Ops & Debug)

Взаимодействие с запущенными контейнерами.

| Команда | Описание |
| :--- | :--- |
| `docker ps` | Список запущенных контейнеров. |
| `docker ps -a` | Список всех контейнеров (включая остановленные). |
| `docker logs <container>` | Показать логи (STDOUT/STDERR). |
| `docker logs -f` | Читать логи в реальном времени (tail -f). |
| `docker exec -it <name> sh` | Запустить интерактивную оболочку внутри контейнера. |
| `docker cp <src> <dest>` | Копирование файлов между хостом и контейнером. |
| `docker inspect <name>` | Показать полную JSON-конфигурацию контейнера. |
| `docker stats` | Показать потребление ресурсов (CPU, RAM) в реальном времени. |
| `docker top <name>` | Показать процессы, работающие внутри контейнера. |

## Images

Работа с образами.

| Команда | Описание |
| :--- | :--- |
| `docker build -t <tag> .` | Собрать образ из Dockerfile в текущей директории. |
| `docker images` | Список локальных образов. |
| `docker pull <image>` | Скачать образ из Registry. |
| `docker push <image>` | Отправить образ в Registry. |
| `docker rmi <image>` | Удалить образ локально. |
| `docker tag <src> <target>` | Создать новый тег для существующего образа. |
| `docker history <image>` | Показать слои, из которых состоит образ. |
| `docker save -o file.tar <img >` | Сохранить образ в tar-архив (для оффлайн переноса). |
| `docker load -i file.tar` | Загрузить образ из tar-архива. |

## Volumes

Управление персистентными данными.

| Команда | Описание |
| :--- | :--- |
| `docker volume create <name>` | Создать именованный том. |
| `docker volume ls` | Список томов. |
| `docker volume inspect` | Показать детали (путь на диске хоста). |
| `docker volume rm <name>` | Удалить том (данные будут потеряны!). |
| `docker volume prune` | Удалить все неиспользуемые тома. |

## Networks

Управление сетевыми интерфейсами.

| Команда | Описание |
| :--- | :--- |
| `docker network create <name>` | Создать пользовательскую сеть (bridge). |
| `docker network ls` | Список сетей. |
| `docker network inspect` | Показать детали (subnet, подключенные контейнеры). |
| `docker network connect <net> <cont>` | Подключить контейнер к сети на лету. |
| `docker network disconnect` | Отключить контейнер от сети. |

## System & Cleanup

Очистка диска и системная информация.

| Команда | Описание |
| :--- | :--- |
| `docker info` | Системная информация (версия ядра, драйверы, кол-во контейнеров). |
| `docker system df` | Показать использование диска объектами Docker. |
| `docker system prune` | Удалить остановленные контейнеры, неиспользуемые сети и dangling images. |
| `docker system prune -a` | Удалить вообще всё, что не используется запущенными контейнерами (включая все образы). |
| `docker system events` | Поток событий демона в реальном времени (полезно для отладки CI). |

## Docker Compose

Команды `docker compose` (v2).

| Команда | Описание |
| :--- | :--- |
| `docker compose up -d` | Собрать, (пере)создать и запустить сервисы в фоне. |
| `docker compose down` | Остановить и удалить контейнеры и сети проекта. |
| `docker compose down -v` | То же + удалить Volumes. |
| `docker compose logs -f` | Логи всех сервисов сразу. |
| `docker compose ps` | Статус сервисов проекта. |
| `docker compose build` | Только собрать образы, без запуска. |
| `docker compose config` | Валидация и просмотр итогового YAML. |
