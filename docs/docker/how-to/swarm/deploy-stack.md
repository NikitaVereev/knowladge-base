---
title: "Развёртывание стеков (Stacks)"
type: how-to
tags: [docker, swarm, stack, deploy, compose, services]
sources:
  docs: "https://docs.docker.com/engine/swarm/stack-deploy/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---


Стек (Stack) — это декларативное описание всего приложения (сервисы, сети, тома, секреты) в одном файле.

## Отличия от Docker Compose

Команда `docker stack deploy` использует стандартный файл `docker-compose.yml`, но:
1.  Игнорирует директиву `build` (образы должны быть заранее собраны и отправлены в Registry).
2.  Использует секцию `deploy` для настроек масштабирования и размещения.

## Пример Production-стека

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - webnet
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure

  api:
    image: my-registry/api:v1
    networks:
      - webnet
      - dbnet
    secrets:
      - db_password
    deploy:
      replicas: 3
      placement:
        constraints: [node.role == worker]

  db:
    image: postgres:14
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - dbnet
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  webnet:
  dbnet:
    internal: true

volumes:
  db-data:

secrets:
  db_password:
    external: true
```

## Основные команды

1.  **Запуск/Обновление:**
    ```bash
    docker stack deploy -c docker-compose.yml myapp
    ```
2.  **Список сервисов:**
    ```bash
    docker stack services myapp
    ```
3.  **Удаление:**
    ```bash
    docker stack rm myapp
    ```



