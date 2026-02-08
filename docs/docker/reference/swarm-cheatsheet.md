---
title: "Docker Swarm — Cheatsheet"
type: reference
tags: [docker, swarm, cli, commands, cheatsheet, best-practices]
sources:
  docs: "https://docs.docker.com/engine/swarm/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/cheatsheet]]"
  - "[[docker/how-to/swarm/deploy-stack]]"
---

# Docker Swarm — Cheatsheet

> **Справочник:** Все команды Docker Swarm CLI + best practices управления кластером.

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
| `docker service ps <n>` | Показать задачи (контейнеры) конкретного сервиса. |
| `docker service scale <n>=5` | Изменить количество реплик. |
| `docker service update --image <img:tag> <n>` | Обновить образ сервиса (Rolling Update). |
| `docker service logs <n>` | Посмотреть логи. |

## Stack Management

| Команда | Описание |
| :--- | :--- |
| `docker stack deploy -c file.yml <n>` | Развернуть стек. |
| `docker stack ls` | Список стеков. |
| `docker stack rm <n>` | Удалить стек. |

## Best Practices

### Менеджеры
*   **Количество:** Всегда нечётное число (1, 3, 5). 3 — золотой стандарт.
*   **Ресурсы:** Не нагружайте менеджеры приложениями. Переведите в `Drain`:
    ```bash
    docker node update --availability drain <manager-node>
    ```

### Безопасность
*   **Секреты:** Используйте `docker secret` вместо ENV переменных.
*   **TLS:** Swarm шифрует управляющий трафик по умолчанию. Включите шифрование данных (`--opt encrypted`) для overlay-сетей.

### Развёртывание
*   **Tags:** Никогда не используйте `:latest` в продакшене. Фиксируйте версии (`:v1.2.3`).
*   **Limits:** Всегда задавайте лимиты ресурсов в `deploy.resources`:
    ```yaml
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
    ```