---
title: "02. Сборка образов"
type: tutorial
tags: [docker, tutorial, dockerfile, nodejs, optimization, security, multi-stage]
sources:
  docs: "https://docs.docker.com/build/building/best-practices/"
related:
  - "[[docker/explanation/images-and-layers]]"
  - "[[docker/how-to/optimize-builds]]"
  - "[[docker/reference/dockerfile]]"
prev: "[[docker/tutorials/01-first-container]]"
next: "[[docker/tutorials/03-compose-app]]"
---

# 2: Dockerfile

> **Цель:** Написать Dockerfile: от наивного (1GB) до production-ready (150MB).
> Освоить multi-stage builds, кэширование слоёв, .dockerignore.

Написать `Dockerfile` — легко. Написать **хороший** `Dockerfile` — искусство.
В этом уроке мы возьмем простое Node.js приложение и пройдем путь от "плохого" образа (800 МБ, root, небезопасно) к "идеальному" (100 МБ, non-root, кэширование).

## Подготовка

Создайте папку `docker-lesson-2` и два файла внутри.

**`package.json`**:
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

**`index.js`**:
```javascript
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello Safe World!'));
app.listen(3000, () => console.log('Server started on 3000'));
```

## 1: "Наивный" подход (Плохо)

Создайте файл `Dockerfile.bad`:

```dockerfile
FROM node
COPY . .
RUN npm install
CMD ["npm", "start"]
```

**Почему это плохо?**
1.  `FROM node`: Это образ весит ~1 ГБ (Debian Full).
2.  `COPY . .`: Любое изменение в коде сбрасывает кэш `npm install`. Сборка будет долгой.
3.  Работает от `root`.
4.  Нет `.dockerignore`: в образ попадут `node_modules` с вашего компа (если они есть) и `.git`.

## 2: Оптимизация слоев и размера

Создайте `.dockerignore`:
```text
node_modules
.git
Dockerfile*
```

Создайте `Dockerfile.better`:

```dockerfile
# Используем Alpine (легкий Linux)
FROM node:18-alpine

WORKDIR /app

# Сначала зависимости (для кэша)
COPY package.json package-lock.json ./
RUN npm install --omit=dev

# Потом код
COPY . .

CMD ["node", "index.js"]
```

**Что улучшили?**
*   Размер образа упал с 1 ГБ до ~150 МБ (Alpine).
*   Кэш: Если вы меняете `index.js`, шаг `npm install` берется из кэша (мгновенно).
*   `CMD ["node"]`: Запуск напрямую (PID 1), сигналы проходят корректно.

## 3: Production Grade

Теперь добавим безопасность (Non-root user) и Multi-stage (если бы у нас была компиляция, но для чистоты примера Node.js это тоже полезно).

Создайте `Dockerfile` (финальный):

```dockerfile
# --- Build Stage ---
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
# Используем ci для строгой установки версий из lock-файла
RUN npm ci

# --- Runtime Stage ---
FROM node:18-alpine

# Создаем пользователя (в Alpine node юзер уже есть, но для примера)
# RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Копируем только нужное из builder-а (или чистые зависимости)
COPY --from=builder /app/node_modules ./node_modules
COPY . .

# Переключаемся на обычного пользователя (Безопасность!)
USER node

EXPOSE 3000

CMD ["node", "index.js"]
```

## Проверка

1.  **Сборка:**
    ```bash
    docker build -t my-app:v1 .
    ```

2.  **Запуск:**
    ```bash
    docker run --rm -p 3000:3000 --name app my-app:v1
    ```

3.  **Проверка безопасности:**
    Откройте новый терминал и проверьте пользователя внутри контейнера:
    ```bash
    docker exec app whoami
    # Вывод: node (ОТЛИЧНО! Не root)
    ```

## Итоги
1.  Весит мало (Alpine).
2.  Собирается быстро (Layer Caching).
3.  Безопасен (Non-root user).
4.  Не содержит мусора (`.dockerignore`).

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `COPY . .` до `npm install` | Каждое изменение кода сбрасывает кэш зависимостей | Сначала `COPY package*.json`, потом `npm ci`, потом `COPY . .` |
| Нет `.dockerignore` | Build context 500MB+, сборка медленная | Добавить `node_modules`, `.git`, `dist` в `.dockerignore` |
| `CMD npm start` (shell form) | SIGTERM не доходит до Node.js, `docker stop` зависает 10 сек | `CMD ["npm", "start"]` (exec form) или tini |
| `FROM node:latest` | Непредсказуемый образ: сегодня Node 20, завтра Node 22 | Фиксировать версию: `FROM node:20-alpine` |
