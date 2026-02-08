---
title: "Docker: Next.js"
type: recipe
tags: [docker, recipe, nextjs, react, node, multi-stage, standalone, production]
sources:
  docs: "https://nextjs.org/docs/app/building-your-application/deploying#docker-image"
  example: "https://github.com/vercel/next.js/tree/canary/examples/with-docker"
related:
  - "[[docker/explanation/images-and-layers]]"
  - "[[docker/how-to/optimize-builds]]"
  - "[[docker/how-to/production-hardening]]"
  - "[[docker/how-to/recipes/nginx-reverse-proxy]]"
  - "[[docker/how-to/recipes/fullstack]]"
---

# Next.js в Docker

> Production-ready Dockerfile для Next.js (App Router / Pages Router).
> Финальный образ: ~150MB, non-root, standalone mode, multi-stage.

## Быстрый старт

```bash
# Скопируй 3 файла (Dockerfile, compose.yaml, .dockerignore) в корень проекта
docker compose up
# → http://localhost:3000
```

## Подготовка: next.config.js

Добавь `output: 'standalone'` — Next.js создаст автономный сервер, который не тянет за собой весь `node_modules`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
}

module.exports = nextConfig
```

## Dockerfile

```dockerfile
# ─── Stage 1: Установка зависимостей ─────────────────────────
FROM node:20-alpine AS deps
# alpine (~50MB) вместо debian (~350MB)
# Зависимость libc6-compat нужна для некоторых npm-пакетов на Alpine
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Копируем ТОЛЬКО файлы зависимостей → кэш слоя сохраняется
# при изменении кода приложения
COPY package.json package-lock.json ./
RUN npm ci
# npm ci вместо npm install:
# - строго следует package-lock.json (воспроизводимая сборка)
# - быстрее (не обновляет lock-файл)
# - удаляет node_modules перед установкой

# ─── Stage 2: Сборка приложения ──────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

# Копируем зависимости из предыдущего стейджа
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Переменные окружения для сборки
# NEXT_TELEMETRY_DISABLED — отключаем телеметрию Vercel
ENV NEXT_TELEMETRY_DISABLED=1

RUN npm run build
# next build создаёт .next/standalone — автономный сервер
# Размер standalone: ~30MB vs ~300MB полного node_modules

# ─── Stage 3: Production runtime ─────────────────────────────
FROM node:20-alpine AS runner
WORKDIR /app

# Production-переменные
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# Non-root пользователь (безопасность)
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Копируем только необходимое из builder:
# 1. public/ — статические файлы (не включены в standalone)
COPY --from=builder /app/public ./public

# 2. standalone — автономный сервер + минимальные node_modules
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./

# 3. static — собранные CSS/JS бандлы (не включены в standalone)
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT=3000
# hostname 0.0.0.0 обязателен для работы внутри контейнера
ENV HOSTNAME="0.0.0.0"

# standalone сервер — один файл server.js
CMD ["node", "server.js"]
```

## compose.yaml — development

```yaml
services:
  app:
    build:
      context: .
      target: deps              # останавливаемся на стейдже deps
    command: npm run dev        # Next.js dev server с HMR
    ports:
      - "3000:3000"
    volumes:
      - .:/app                  # live reload — изменения в коде видны сразу
      - /app/node_modules       # anonymous volume — не перезаписывать node_modules с хоста
      - /app/.next              # anonymous volume — кэш сборки внутри контейнера
    environment:
      - NODE_ENV=development
      - WATCHPACK_POLLING=true  # hot reload в Docker (inotify не работает через bind mount)
```

## compose.yaml — production

```yaml
services:
  app:
    build:
      context: .
      target: runner           # финальный production стейдж
    ports:
      - "127.0.0.1:3000:3000" # только localhost (Nginx проксирует снаружи)
    read_only: true
    tmpfs:
      - /app/.next/cache:size=200m  # кэш ISR/image optimization
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## .dockerignore

```
node_modules
.next
.git
.gitignore
README.md
Dockerfile
docker-compose*.yml
compose*.yaml
.env*.local
.vscode
.idea
*.md
```

## Почему именно так

### Standalone output
Без `output: 'standalone'` в образ попадает весь `node_modules` (~300MB). Standalone трейсит зависимости и создаёт минимальный сервер (~30MB). Это **ключевая** настройка.

### Три стейджа, а не два
- `deps` — устанавливаем зависимости (кэшируется отдельно от кода)
- `builder` — собираем приложение (пересобирается при изменении кода, но зависимости берутся из кэша)
- `runner` — только runtime (нет npm, нет исходников, нет devDependencies)

### COPY public и static отдельно
Standalone НЕ включает `public/` и `.next/static/`. Без этих строк:
- Не загрузятся шрифты, favicon, изображения из `public/`
- Не загрузятся CSS и JS бандлы

### HOSTNAME="0.0.0.0"
По умолчанию Next.js слушает `localhost`, что внутри контейнера означает только loopback-интерфейс. Docker перенаправляет трафик на IP контейнера, а не на localhost. Без этой переменной приложение недоступно снаружи контейнера.

### WATCHPACK_POLLING для dev
Файловая система Docker (bind mount) не поддерживает inotify на macOS/Windows. Без polling hot reload не работает.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Нет `output: 'standalone'` в next.config.js | Образ 1.5GB, или `server.js` не найден | Добавить `output: 'standalone'` |
| Не скопирован `public/` | 404 на favicon, шрифты, изображения | `COPY --from=builder /app/public ./public` |
| Не скопирован `.next/static` | Страница без CSS/JS (белый экран) | `COPY --from=builder /app/.next/static ./.next/static` |
| Нет `HOSTNAME="0.0.0.0"` | `curl: (7) Failed to connect` | Добавить `ENV HOSTNAME="0.0.0.0"` |
| `NEXT_PUBLIC_*` не видны в runtime | Переменные `undefined` в браузере | `NEXT_PUBLIC_*` инлайнятся при **сборке** — передавать как `--build-arg` |
| Sharp не работает на Alpine | Ошибка при image optimization | `RUN npm install --os=linux --cpu=arm64 sharp` (для arm) или отключить image optimization |
| Hot reload не работает в dev | Изменения не видны без рестарта | Добавить `WATCHPACK_POLLING=true` |

## Переменные окружения

Next.js различает два типа переменных:

```yaml
# Build-time (вшиваются в бандл при сборке)
# Для NEXT_PUBLIC_* — передавать через --build-arg
build:
  args:
    - NEXT_PUBLIC_API_URL=https://api.example.com

# Runtime (доступны только на сервере)
environment:
  - DATABASE_URL=postgresql://...
  - API_SECRET=...
```

## Варианты

- С Nginx: поставить [[docker/how-to/recipes/nginx-reverse-proxy]] перед Next.js для SSL и кэширования
- Полный стек: [[docker/how-to/recipes/fullstack]] — Next.js + API + PostgreSQL + Redis
- Monorepo (Turborepo): добавить `turbo prune` перед `npm ci` в стейдже deps
