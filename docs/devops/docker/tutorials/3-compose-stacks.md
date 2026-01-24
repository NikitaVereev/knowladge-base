---
title: "3 Популярные стеки Compose"
description: "Готовые рецепты (boilerplate) для быстрого старта: Node.js + Mongo и Python + Redis + Worker."
---

Ниже приведены готовые шаблоны `compose.yaml`, которые можно копировать и использовать как базу для своих проектов.

## 1. MERN Stack (Node.js + MongoDB)

Классическая связка API и базы данных.

```yaml
services:
  api:
    build: .
    # Пробрасываем порт для доступа с хоста
    ports: 
      - "3000:3000"
    environment:
      # Важно: хост базы данных равен имени сервиса 'mongo'
      - MONGO_URI=mongodb://mongo:27017/myapp
    depends_on:
      - mongo
    # Hot Reload для разработки (если используете nodemon)
    develop:
      watch:
        - action: sync
          path: .
          target: /app
          ignore:
            - node_modules/

  mongo:
    image: mongo:6
    restart: always
    # Данные базы храним в именованном томе
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

## 2. Python (Flask/FastAPI) + Redis Worker

Асинхронная обработка задач (очереди). Здесь у нас один и тот же код (образ) используется для двух разных целей: веб-сервер и фоновый воркер.

```yaml
services:
  web:
    build: .
    command: python app.py
    ports: 
      - "8000:8000"
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis

  worker:
    build: .
    # Переопределяем команду запуска для воркера
    command: python worker.py 
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis
    # Воркеров можно масштабировать: docker compose up -d --scale worker=3
  
  redis:
    image: redis:alpine
    # Redis работает только внутри сети, наружу не торчит (безопасно)
```

## 3. Fullstack (Frontend + Backend + DB)

Пример разделения клиентской и серверной части.

```yaml
services:
  frontend:
    build: ./frontend
    ports: ["80:80"] # Nginx отдает статику
    depends_on:
      - backend

  backend:
    build: ./backend
    expose: ["3000"] # Порт доступен только frontend-у (через proxy_pass)
    environment:
      - DB_HOST=postgres
  
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pg_data:/var/lib/postgresql/data

volumes:
  pg_data:
```
