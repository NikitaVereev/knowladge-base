---
title: "Продвинутый Docker Compose"
description: "Использование Profiles, Override-файлов, переменных окружения и YAML-якорей."
---


## Профили (Profiles)
Профили позволяют запускать только нужную часть сервисов. Это удобно для монорепозиториев или сложных систем, где не всегда нужны все компоненты (например, workers или мониторинг).

```yaml
services:
  web:
    image: nginx
    # Запускается всегда (без профиля)

  worker:
    image: my-worker
    profiles: ["workers"]  # Только с --profile workers

  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]
```

**Запуск:**
```bash
docker compose up                      # Только web
docker compose --profile workers up    # web + worker
```

## Переопределение конфигураций (Override)
Docker Compose умеет объединять несколько файлов. По умолчанию он ищет `docker-compose.yml` и `docker-compose.override.yml`.

### Сценарий использования
1. **`docker-compose.yml`** — Базовая конфигурация (образы, порты, связи).
2. **`docker-compose.override.yml`** — Локальные переопределения для разработки (bind mounts, dev-порты, environment). Не коммитится в Git.
3. **`docker-compose.prod.yml`** — Конфигурация для продакшена (restart policy, limits).

**Пример запуска продакшена:**
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## DRY (Don't Repeat Yourself) с YAML якорями
Если у вас много похожих сервисов (например, воркеры с одинаковыми env-vars), используйте якоря.

```yaml
x-common-env: &common_env
  environment:
    DB_HOST: postgres
    REDIS_HOST: redis

services:
  api:
    <<: *common_env
    image: my-api

  worker:
    <<: *common_env
    image: my-worker
```

## Связанные материалы
- [[devops/docker/how-to/compose-best-practices|Best Practices для Compose]]
- [[devops/docker/tutorials/microservices-compose|Пример микросервисной архитектуры]]
