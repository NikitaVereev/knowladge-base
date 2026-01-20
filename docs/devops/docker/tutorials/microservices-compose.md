---
title: "Микросервисы в Docker Compose"
description: "Пример конфигурации сложной системы: API Gateway, сервисы, RabbitMQ, мониторинг."
---


Пример файла для системы с несколькими сервисами, очередью и мониторингом, разбитый по профилям.

```yaml
services:
  # --- Infrastructure ---
  db:
    image: postgres:15-alpine
    profiles: ["core"]

  rabbitmq:
    image: rabbitmq:3-management
    profiles: ["core"]
    ports: ["15672:15672"]

  # --- Application ---
  api-gateway:
    image: nginx
    depends_on: [auth-service, product-service]
    ports: ["80:80"]
    profiles: ["core"]

  auth-service:
    build: ./auth
    depends_on:
      db:
        condition: service_healthy
    profiles: ["core"]

  product-service:
    build: ./products
    depends_on:
      db:
        condition: service_healthy
    profiles: ["core"]

  # --- Background Workers ---
  email-worker:
    build: ./workers/email
    depends_on: [rabbitmq]
    profiles: ["workers"]

  # --- Monitoring ---
  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]
```

## Как с этим работать

1. **Запуск основного приложения:**
   ```bash
   # Запустятся db, rabbitmq, gateway и сервисы (профиль "core" тут условен, обычно это default)
   COMPOSE_PROFILES=core docker compose up
   ```

2. **Запуск с воркерами:**
   ```bash
   docker compose --profile core --profile workers up
   ```

3. **Отладка мониторинга:**
   ```bash
   docker compose --profile monitoring up
   ```

## Связанные материалы
- [[devops/docker/explanation/compose-advanced|Профили и переопределения]]
