---
title: "1 Docker CLI commands"
description: "Шпаргалка по основным командам Docker (Engine & Compose), актуальная для версии 2026 года."
---

## Контейнеры (Управление)

| Команда | Описание |
| :--- | :--- |
| `docker run -d --name <name> <img:tag>` | Запустить новый контейнер в фоне. |
| `docker ps` | Показать **только запущенные** контейнеры. |
| `docker ps -a` | Показать **все** (в том числе остановленные). |
| `docker stop <name>` | Корректная остановка (SIGTERM). |
| `docker rm <name>` | Удалить остановленный контейнер. |
| `docker rm -f <name>` | Force-удаление (работающего) контейнера. |
| `docker container prune` | Удалить **ВСЕ** остановленные контейнеры разом. |

## Отладка и Инспекция

| Команда | Описание |
| :--- | :--- |
| `docker logs -f <name>` | Читать логи в реальном времени (tail -f). |
| `docker exec -it <name> sh` | Зайти внутрь контейнера (интерактивная консоль). |
| `docker inspect <name>` | Показать полную инфу (IP, Volumes, Env) в JSON. |
| `docker stats` | Мониторинг ресурсов (CPU/RAM) в реальном времени. |
| `docker port <name>` | Показать проброшенные порты. |

## Образы (Images)

| Команда | Описание |
| :--- | :--- |
| `docker images` | Список локальных образов. |
| `docker pull <img:tag>` | Скачать образ из реестра. |
| `docker build -t <img:tag> .` | Собрать образ из Dockerfile. |
| `docker rmi <img:tag>` | Удалить образ с диска. |
| `docker image prune -a` | Удалить **все** образы, не используемые контейнерами (чистка места). |
| `docker save -o file.tar <img>` | Экспорт образа в файл (для offline-переноса). |
| `docker load -i file.tar` | Импорт образа из файла. |

## Docker Compose

| Команда | Описание |
| :--- | :--- |
| `docker compose up -d` | Поднять стек в фоне. |
| `docker compose up --build -d` | Поднять с принудительной пересборкой. |
| `docker compose down` | Остановить и удалить контейнеры и сеть. |
| `docker compose down -v` | **Осторожно!** Удалить контейнеры, сеть и **тома (volumes)** с данными. |
| `docker compose logs -f` | Логи всего стека. |
| `docker compose logs -f <service>` | Логи конкретного сервиса. |
| `docker compose restart <service>` | Перезапустить отдельный сервис. |

## Очистка системы (Housekeeping)

| Команда | Описание |
| :--- | :--- |
| `docker system prune` | Удалить остановленные контейнеры, неиспользуемые сети и dangling-образы. |
| `docker system prune -a` | Полная очистка: удалит вообще всё, что не запущено прямо сейчас (включая все скачанные образы). |
| `docker system df` | Показать, сколько места на диске занимает Docker. |
