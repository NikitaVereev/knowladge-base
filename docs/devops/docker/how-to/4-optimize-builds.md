---
title: 4 Оптимизация сборки (BuildKit)
type: how-to
tags: [docker, build, buildkit, cache, multi-stage, arm64, ci-cd]
---

# Оптимизация сборки образов

Скорость сборки напрямую влияет на Time-to-Market и время ожидания CI/CD пайплайнов. В этом руководстве мы рассмотрим современные техники ускорения билдов с использованием **BuildKit**.

## 1. Включение BuildKit

BuildKit — это современный движок сборки, который работает параллельно, кэширует эффективнее и поддерживает секреты.

В новых версиях Docker Desktop и Linux (23.0+) он включен по умолчанию.
Если нет, включите принудительно:

```bash
export DOCKER_BUILDKIT=1
docker build .
```

## 2. Кэширование слоев (Layer Caching)

Docker кэширует каждый шаг `RUN`, `COPY`. Если файл не изменился, Docker берет слой из кэша.

### Стратегия: "Сначала зависимости"
Самая частая ошибка — копирование всего кода сразу.

**❌ Плохо (Кэш сбрасывается при каждом изменении кода):**
```dockerfile
COPY . .
RUN npm ci
```

**✅ Хорошо (Кэш `npm ci` живет, пока не изменится `package.json`):**
```dockerfile
COPY package.json package-lock.json ./
RUN npm ci

COPY . .
# Дальше сборка приложения...
```

## 3. Mount Caches (Кэш компилятора)

Даже если слой `RUN npm ci` инвалидировался (вы добавили одну библиотеку), BuildKit позволяет не качать *все* библиотеки заново, а использовать локальный кэш пакетного менеджера (pip, npm, go, maven).

Используйте синтаксис `--mount=type=cache`:

**Node.js:**
```dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

**Go:**
```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o app .
```

**Apt (Debian/Ubuntu):**
```dockerfile
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y gcc
```

Этот кэш сохраняется между разными сборками (`docker build`), даже если слои меняются.

## 4. Multi-Stage Builds (Уменьшение размера)

Не тащите компиляторы и исходный код в продакшн.

### Пример 1: Go (Static Binary)
```dockerfile
# Stage 1: Build
FROM golang:1.21-stretch AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# CGO_ENABLED=0 для статической линковки
RUN CGO_ENABLED=0 GOOS=linux go build -o /bin/app main.go

# Stage 2: Runtime (Scratch - пустой образ)
FROM scratch
# Копируем сертификаты для HTTPS запросов (в scratch их нет)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /bin/app /app
CMD ["/app"]
```
*Результат: Образ весит ~10-15MB.*

### Пример 2: Node.js (Frontend / React)
```dockerfile
# Stage 1: Build Frontend
FROM node:18-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
# На выходе получаем папку /app/dist (или /build)

# Stage 2: Serve with Nginx
FROM nginx:alpine
# Копируем только статику
COPY --from=builder /app/dist /usr/share/nginx/html
# Копируем кастомный конфиг (если нужен)
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
*Результат: Легкий Nginx вместо тяжелого Node.js процесса.*

### Пример 3: Node.js (Backend / NestJS)
Для бэкенда тоже нужен multi-stage, чтобы не тащить `devDependencies` (TypeScript компилятор) в прод.

```dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production Dependencies Only
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
# Ставим ТОЛЬКО prod зависимости
RUN npm ci --omit=dev 

# Stage 3: Final Image
FROM node:18-alpine
WORKDIR /app
COPY --from=production /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/main.js"]
```

## 5. dockerignore (Контекст сборки)

Перед началом сборки Docker CLI отправляет демону все файлы из текущей папки (Build Context). Если у вас там лежит `.git` (200MB) или `node_modules` (500MB), сборка будет начинаться с долгой паузы "Sending build context to Docker daemon".

Создайте `.dockerignore`:
```text
.git
node_modules
dist
build
coverage
*.log
.env
```

## 6. Multi-Platform Builds (ARM64 + AMD64)

Для сборки под Apple Silicon (M1/M2) и серверы (Intel/AMD) используйте `docker buildx`.

1.  **Создайте билдер:**
    ```bash
    docker buildx create --use
    ```

2.  **Соберите мульти-архитектурный образ:**
    ```bash
    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t my-app:latest \
      --push .
    ```
    *Флаг `--push` обязателен, так как локальный Docker не может хранить мульти-архитектурный манифест в обычном списке `docker images`. Образ улетит сразу в Registry.*

## Резюме: Чек-лист оптимизации

1.  [ ] **.dockerignore** настроен.
2.  [ ] Зависимости копируются **до** исходного кода.
3.  [ ] Используется **Multi-stage** (отдельно build, отдельно runtime).
4.  [ ] Добавлены **Mount Caches** (`--mount=type=cache`) для `npm`/`go`/`apt`.
5.  [ ] Используется **BuildKit** (вывод сборки цветной и параллельный).
