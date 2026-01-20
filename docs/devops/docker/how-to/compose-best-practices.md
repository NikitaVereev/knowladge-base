---
title: "Docker Compose Best Practices"
description: "Советы по организации docker-compose файлов, healthcheck-и и управление ресурсами."
---


## 1. Используйте Healthchecks и `depends_on`
Просто `depends_on` ждет только запуска контейнера, но не готовности приложения. Используйте `condition: service_healthy`.

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    depends_on:
      db:
        condition: service_healthy
```

## 2. Разделяйте Dev и Prod окружения
Не пытайтесь уместить всё в одном файле с кучей `if`. Используйте **Override-файлы**.
* В `docker-compose.yml` оставьте общую структуру.
* В `docker-compose.override.yml` добавьте `build: .` и `volumes: [".:/app"]` для разработки.
* В `docker-compose.prod.yml` добавьте `restart: always` и уберите bind-mounts кода.

## 3. Масштабирование (Scaling)
Compose позволяет запускать несколько экземпляров одного сервиса (если не заняты порты хоста).

```bash
docker compose up --scale worker=3
```
*Совет: Не используйте `container_name` для масштабируемых сервисов, иначе будет конфликт имен.*

## 4. Ограничение ресурсов
Для продакшена всегда задавайте лимиты.

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

## Связанные материалы
- [[devops/docker/explanation/compose-advanced|Продвинутые возможности Compose]]
