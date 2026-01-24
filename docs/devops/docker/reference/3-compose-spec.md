---
title: "3 Спецификация Docker Compose"
description: "Шпаргалка по полям `compose.yaml`: Services, Volumes, Networks, Healthcheck и лимиты ресурсов."
---

## Полный пример `compose.yaml`

```yaml
# version: "3.8"  <-- Больше не нужно в V2

name: my-project  # Имя проекта (влияет на префикс контейнеров/сетей)

services:
  web:
    image: nginx:alpine
    container_name: my-web-custom-name  # Лучше не использовать, если планируете scale
    build:
      context: .              # Папка сборки
      dockerfile: Dockerfile  # Имя файла
      args:
        APP_ENV: production
      target: production      # Для multi-stage билдов
    
    ports:
      - "8080:80"             # Host:Container
    
    environment:
      - NODE_ENV=production
      - DB_HOST=db
    env_file:                 # Загрузка переменных из файла
      - .env

    volumes:
      - ./data:/app/data:ro   # Bind Mount (Read Only)
      - db_data:/var/lib/mysql # Named Volume

    depends_on:
      db:
        condition: service_healthy # Ждать, пока база реально не заработает (а не просто запустится)
      redis:
        condition: service_started

    networks:
      - backend
      - frontend

    restart: unless-stopped   # Политика перезапуска

    # Лимиты ресурсов (Работает и без Swarm в V2)
    deploy:
      resources:
        limits:
          cpus: '0.50'        # 50% ядра
          memory: 512M        # Максимум RAM

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - backend

# Объявление томов
volumes:
  db_data:
    driver: local

# Объявление сетей
networks:
  backend:
    internal: true  # Сеть без доступа в интернет (только внутри)
  frontend:
    driver: bridge
```

## Ключевые поля

### `build`
Описание того, как собирать образ, если он не скачивается.
```yaml
build:
  context: ./backend
  dockerfile: Dockerfile.dev
```

### `depends_on`
Определяет порядок запуска.
*   `condition: service_started` (по умолчанию): Ждет только запуска контейнера.
*   `condition: service_healthy`: Ждет, пока `healthcheck` не вернет "healthy". **Используйте для баз данных.**

### `healthcheck`
Команда внутри контейнера, которая сообщает Docker, живо ли приложение.
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
```

### `deploy.resources`
В старых версиях лимиты задавались через `mem_limit`. В современной спецификации (v3 / Compose Spec) это делается через секцию `deploy`.
> **Важно:** В Docker Compose V2 секция `deploy` работает и при обычном запуске (`docker compose up`), а не только в режиме Swarm.

### `ports` vs `expose`
*   **`ports`**: `- "8080:80"` — Открывает порт наружу (на хост-машину).
*   **`expose`**: `- "80"` — Просто делает порт доступным для *других контейнеров* (и для линковки), но не пробрасывает на хост.
