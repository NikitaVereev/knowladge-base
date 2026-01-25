---
title: "6 Микросервисы - Профили и Монорепозиторий"
description: "Как управлять сложной архитектурой (Backend, Frontend, Workers, Tests) в одном docker-compose.yml с помощью Profiles."
---

В монорепозиториях часто возникает проблема: `docker compose up` запускает 20 контейнеров, хотя вам нужно поработать только над фронтендом. Решение — **Profiles** (доступно в Docker Compose V2).

## Структура проекта
```text
/my-project
├── docker-compose.yml
├── backend/
│   └── Dockerfile
├── frontend/
│   └── Dockerfile
└── e2e-tests/
    └── cypress.json
```

## Конфигурация (docker-compose.yml)

Мы разделим сервисы на группы с помощью поля `profiles`.

```yaml
services:
  # --- INFRASTRUCTURE (Запускается всегда) ---
  # Сервисы без профилей стартуют всегда, если не указано иное.
  db:
    image: postgres:15
    volumes: [pg_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
  
  redis:
    image: redis:alpine

  # --- CORE (Основное приложение) ---
  backend:
    build: ./backend
    profiles: ["core", "api"]  # Входит в профили core И api
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  frontend:
    build: ./frontend
    profiles: ["core", "ui"]   # Входит в профили core И ui
    ports: ["3000:80"]

  # --- WORKERS (Фоновые задачи) ---
  worker-mail:
    build: ./backend
    command: python worker_mail.py
    profiles: ["async"]        # Только для профиля async
    depends_on: [redis]

  worker-reports:
    build: ./backend
    command: python worker_reports.py
    profiles: ["async"]

  # --- TESTING (Тесты) ---
  cypress:
    image: cypress/included:13.6.0
    profiles: ["test"]         # Запускать только для тестов
    environment:
      - CYPRESS_baseUrl=http://frontend:80
    depends_on:
      - frontend
      - backend

volumes:
  pg_data:
```

## Как с этим работать?

### Сценарий 1: Я Frontend-разработчик
Мне не нужны тяжелые воркеры, только API и UI.
```bash
docker compose --profile core up
# Запустит: db, redis, backend, frontend
```

### Сценарий 2: Я работаю над бэкендом и очередями
Мне не нужен UI, но нужны воркеры.
```bash
docker compose --profile api --profile async up
# Запустит: db, redis, backend, worker-mail, worker-reports
```

### Сценарий 3: Запуск E2E тестов в CI
Мне нужно поднять всё приложение и прогнать тесты, а потом всё погасить.
```bash
# --abort-on-container-exit остановит всё, когда cypress закончит работу
docker compose --profile test up --abort-on-container-exit
```

### Сценарий 4: Запустить всё вообще
```bash
docker compose --profile "*" up
```

## Полезный трюк (.env)
Чтобы не писать `--profile` каждый раз, можно задать переменную окружения в файле `.env`:

```ini
# .env
COMPOSE_PROFILES=core,async
```
Теперь `docker compose up` будет автоматически запускать профили `core` и `async`.
