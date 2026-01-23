---
title: "1 Справочник команд Swarm CLI"
description: "Шпаргалка по основным командам управления."
---

## Cluster Management

| Команда | Описание |
| :--- | :--- |
| `docker swarm init` | Инициализировать кластер (текущая нода станет лидером). |
| `docker swarm join` | Присоединить ноду к кластеру (нужен токен). |
| `docker swarm join-token [worker/manager]` | Показать токен для присоединения. |
| `docker node ls` | Список всех нод. |
| `docker node update --availability drain <id>` | Вывести ноду из эксплуатации (убрать задачи). |

## Service Management

| Команда | Описание |
| :--- | :--- |
| `docker service create` | Создать новый сервис. |
| `docker service ls` | Список сервисов. |
| `docker service ps <name>` | Показать задачи (контейнеры) конкретного сервиса. |
| `docker service scale <name>=5` | Изменить количество реплик. |
| `docker service update --image <img:tag> <name>` | Обновить образ сервиса (Rolling Update). |
| `docker service logs <name>` | Посмотреть логи. |

## Stack Management

| Команда | Описание |
| :--- | :--- |
| `docker stack deploy -c file.yml <name>` | Развернуть стек. |
| `docker stack ls` | Список стеков. |
| `docker stack rm <name>` | Удалить стек. |

## Связанные материалы
- [[docs/devops/docker-swarm/explanation/1-architecture|Архитектура Swarm]]
