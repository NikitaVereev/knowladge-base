---
title: 1 CLI Cheat Sheet
description: Краткий справочник команд Docker CLI, сгруппированный по объектам управления.
---

## Containers (Жизненный цикл)

| Команда | Описание |
| :--- | :--- |
| `docker run [opts] image [cmd]` | Создать и запустить новый контейнер |
| `docker create [opts] image` | Создать, но не запускать (статус Created) |
| `docker start container` | Запустить остановленный контейнер |
| `docker stop container` | Graceful stop (SIGTERM + 10s timeout) |
| `docker kill container` | Force stop (SIGKILL) |
| `docker restart container` | Перезапуск (stop + start) |
| `docker pause/unpause` | Заморозить процесс (cgroups freeze) |
| `docker rm container` | Удалить (добавить `-f` для работающего) |
| `docker container prune` | Удалить все остановленные контейнеры |

## Containers (Взаимодействие)

| Команда | Описание |
| :--- | :--- |
| `docker ps` | Список запущенных |
| `docker ps -a` | Список всех (включая остановленные) |
| `docker logs [-f] container` | Вывод логов (stdout/stderr) |
| `docker exec -it container sh` | Запустить доп. процесс внутри (shell) |
| `docker cp src dest` | Копирование файлов (host <-> container) |
| `docker stats` | Live мониторинг ресурсов |
| `docker inspect container` | Полные метаданные (JSON) |
| `docker diff container` | Список измененных файлов |

## Images (Образы)

| Команда | Описание |
| :--- | :--- |
| `docker build -t tag .` | Сборка из Dockerfile |
| `docker images` | Список локальных образов |
| `docker pull image` | Скачать из Registry |
| `docker push image` | Загрузить в Registry |
| `docker rmi image` | Удалить образ |
| `docker image prune` | Удалить dangling (<none>) образы |
| `docker history image` | Показать слои и их размер |
| `docker save/load` | Экспорт/Импорт в tar-архив |
| `docker tag src target` | Создать алиас (новый тег) |

## Volumes & Networks

| Команда | Описание |
| :--- | :--- |
| `docker volume ls` | Список томов |
| `docker volume create name` | Создать том вручную |
| `docker volume rm name` | Удалить том |
| `docker volume prune` | Удалить неиспользуемые тома |
| `docker network ls` | Список сетей |
| `docker network create name` | Создать сеть (bridge по умолчанию) |
| `docker network inspect net` | Показать подключенные контейнеры |
| `docker network connect net cont` | Подключить работающий контейнер к сети |

## System (Docker Engine)

| Команда | Описание |
| :--- | :--- |
| `docker login` | Аутентификация в Registry |
| `docker info` | Информация о системе (ядро, драйверы) |
| `docker version` | Версии Client и Server |
| `docker system df` | Использование диска (images, cont, vols) |
| `docker system prune` | Удалить всё остановленное и неиспользуемое |
| `docker system events` | Лог событий демона в реальном времени |
