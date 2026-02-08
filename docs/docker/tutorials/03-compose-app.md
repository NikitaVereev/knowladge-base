---
title: "03. Docker Compose — полное приложение"
type: tutorial
tags: [docker, tutorial, compose, depends_on, healthcheck, redis, database]
sources:
  docs: "https://docs.docker.com/compose/gettingstarted/"
related:
  - "[[docker/explanation/compose-model]]"
  - "[[docker/explanation/configuration]]"
  - "[[docker/reference/compose-spec]]"
prev: "[[docker/tutorials/02-building-images]]"
next: "[[docker/tutorials/04-networking-workshop]]"
---

# 3: Полноценная среда разработки с Docker Compose

> **Цель:** Запустить App + PostgreSQL + Redis через Docker Compose.
> Освоить healthcheck, depends_on, volumes для данных, hot reload для кода.

Запускать контейнеры по одному (`docker run`) неудобно, когда их становится много. В этом уроке мы создадим реалистичный стек: **Node.js приложение**, которое хранит счетчик просмотров в **Redis**.

Мы научимся связывать контейнеры, управлять зависимостями и использовать "Горячую перезагрузку" (Hot Reload) без пересборки образов.

## Подготовка

В папке `docker-lesson-3`:

1.  **`package.json`**:
    ```json
    {
      "dependencies": {
        "express": "^4.18.2",
        "redis": "^4.6.7"
      }
    }
    ```

2.  **`index.js`**:
    ```javascript
    const express = require('express');
    const { createClient } = require('redis');

    const app = express();
    // Обращаемся к Redis по имени сервиса в Compose: "redis-db"
    const client = createClient({ url: 'redis://redis-db:6379' });

    client.on('error', err => console.log('Redis Client Error', err));

    app.get('/', async (req, res) => {
      const count = await client.incr('hits');
      res.send(`Hello! This page has been viewed ${count} times.`);
    });

    const start = async () => {
      await client.connect();
      app.listen(3000, () => console.log('Server running on 3000'));
    };

    start();
    ```

3.  **`Dockerfile`** (для разработки):
    ```dockerfile
    FROM node:18-alpine
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    # Код НЕ копируем, мы его примонтируем через Compose!
    CMD ["node", "index.js"]
    ```

## Магия Compose

Создайте файл `compose.yaml`:

```yaml
services:
  # Наше приложение
  web:
    build: .
    ports:
      - "3000:3000"
    volumes:
      # Hot Reload: монтируем текущую папку внутрь контейнера
      - ./:/app
      # Важно: исключаем node_modules, чтобы использовать те, что внутри контейнера (Linux), 
      # а не те, что на хосте (Windows/Mac)
      - /app/node_modules
    depends_on:
      redis-db:
        condition: service_healthy
    environment:
      - NODE_ENV=development

  # База данных
  redis-db:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

## Запуск и Работа

1.  **Поднимаем стенд:**
    ```bash
    docker compose up
    # Если хотите в фоне: docker compose up -d
    ```

2.  **Проверка:**
    Откройте `http://localhost:3000`. Вы увидите: "This page has been viewed 1 times".
    Обновите страницу: "... 2 times".
    *Это значит, что Node.js успешно соединился с Redis.*

3.  **Hot Reload (Проверка Bind Mount):**
    Не останавливая контейнеры, откройте `index.js` в редакторе и измените текст:
    `res.send('UPDATED! View count: ${count}');`
    Сохраните файл.

    Вам придется перезапустить процесс Node.js, чтобы подхватить изменения (так как мы не используем nodemon).
    ```bash
    docker compose restart web
    ```
    Обновите браузер. Текст изменился! Мы изменили код в контейнере, не пересобирая образ.

## Уборка

Чтобы удалить все контейнеры и сеть проекта:

```bash
docker compose down
```

## Итоги
1.  **Service Discovery:** Контейнеры видят друг друга по именам (`redis-db`).
2.  **Depends On + Healthcheck:** `web` не запустился, пока `redis` не сказал "PONG". Это предотвращает ошибки подключения при старте.
3.  **Volumes Development:** Мы редактируем код на хосте, а выполняется он в контейнере.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| App стартует раньше БД | `ECONNREFUSED` при подключении к PostgreSQL | `depends_on: db: condition: service_healthy` |
| `docker compose down` удалил данные | «Где моя база?» | Данные в named volume выживают. `-v` удаляет volumes (осторожно!) |
| Порт уже занят | `bind: address already in use` | Другой контейнер или процесс на этом порту. `docker ps` или `lsof -i :5432` |
| Изменения в коде не видны | Hot reload не работает | Проверить: volume примонтирован? `WATCHPACK_POLLING=true` для Next.js? |