---
title: "Примеры стеков Docker Compose"
description: "Готовые рецепты docker-compose.yml для популярных стеков: Node+Mongo, Python+Postgres+Redis."
---


## 1. Node.js + MongoDB

Простой стек для веб-приложения с базой данных. Включает bind mount для разработки (hot reload).

```yaml
services:
  app:
    build: .             # Сборка из текущей директории
    ports:
      - "3000:3000"
    environment:
      # Обращаемся к mongo по имени сервиса 'db'
      MONGO_URL: mongodb://db:27017/myapp
    volumes:
      - .:/app           # Монтируем код для разработки
      - /app/node_modules # Исключаем node_modules
    depends_on:
      - db
  
  db:
    image: mongo:latest
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

## 2. Python + Postgres + Redis

Более сложный стек с базой данных и кэшем. Использует `healthcheck` для правильного порядка запуска.

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy  # Ждать готовности БД
      redis:
        condition: service_started
    environment:
      DB_HOST: db
      REDIS_HOST: redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine

volumes:
  pg_data:
```

## Связанные материалы
- [[5-compose|Введение в Compose]]
- [[devops/docker/how-to/manage-compose-apps|Команды управления]]
