---
title: "Docker: Go"
type: recipe
tags: [docker, recipe, go, golang, scratch, distroless, multi-stage, production]
sources:
  docs: "https://docs.docker.com/language/golang/"
  example: "https://github.com/docker/awesome-compose/tree/master/nginx-golang"
related:
  - "[[docker/explanation/images-and-layers]]"
  - "[[docker/how-to/optimize-builds]]"
  - "[[docker/how-to/production-hardening]]"
---

# Go в Docker

> Минимальный production-образ для Go-приложения.
> Финальный образ: **~10MB** (scratch) или **~30MB** (distroless), non-root.
> Go компилируется в статический бинарник — в runtime не нужны ни ОС, ни зависимости.

## Быстрый старт

```bash
docker compose up -d
# → http://localhost:8080
```

## Dockerfile

```dockerfile
# ─── Stage 1: Сборка ───
FROM golang:1.22-alpine AS builder
WORKDIR /build

# Зависимости (кэшируются отдельно от кода)
COPY go.mod go.sum ./
RUN go mod download

# Код приложения
COPY . .

# Компилируем статический бинарник
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app ./cmd/server
# CGO_ENABLED=0: чистый Go без C-зависимостей → можно запустить на scratch
# -ldflags="-s -w": убираем символы отладки → бинарник на 30% меньше
# -o /app: выходной файл

# ─── Stage 2: Production ───
FROM scratch
# scratch = пустой образ (0 байт). Ни shell, ни libc, ни утилит.
# Можно запустить только статический бинарник.

# Копируем SSL-сертификаты (нужны для HTTPS-запросов наружу)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Non-root: в scratch нет /etc/passwd, но можно задать UID напрямую
COPY --from=builder --chown=65534:65534 /app /app
USER 65534:65534
# 65534 = nobody (стандартный non-root UID)

EXPOSE 8080

# Нет shell → нельзя использовать CMD ["sh", "-c", ...]
# Только exec-форма
ENTRYPOINT ["/app"]
```

### Вариант: с distroless (если нужен debug)

```dockerfile
# scratch не содержит ничего — даже ls и sh для дебага.
# distroless — компромисс: минимальная ОС (~20MB), но есть ca-certs и timezone
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

## compose.yaml — development

```yaml
services:
  app:
    build:
      context: .
      target: builder
    command: go run ./cmd/server
    ports:
      - "8080:8080"
    volumes:
      - .:/build
    environment:
      - ENV=development
```

Для hot-reload в dev — использовать [Air](https://github.com/air-verse/air):

```yaml
  app:
    build:
      context: .
      target: builder
    command: air -c .air.toml
    ports:
      - "8080:8080"
    volumes:
      - .:/build
```

## compose.yaml — production

```yaml
services:
  app:
    build: .
    ports:
      - "127.0.0.1:8080:8080"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 128M
    healthcheck:
      # scratch не содержит wget/curl — healthcheck через Go
      test: ["NONE"]       # использовать /health endpoint через Docker inspect
      # Или собрать отдельный health-бинарник — см. ниже
    restart: unless-stopped
```

### Healthcheck для scratch

В scratch нет утилит. Варианты:

```dockerfile
# Вариант 1: Собрать отдельный бинарник для healthcheck
RUN CGO_ENABLED=0 go build -o /healthcheck ./cmd/healthcheck
# В финальном образе:
COPY --from=builder /healthcheck /healthcheck
HEALTHCHECK CMD ["/healthcheck"]

# Вариант 2: Использовать distroless вместо scratch (есть wget)
# Вариант 3: Проверять через Docker engine (без HEALTHCHECK в Dockerfile)
```

## .dockerignore

```
.git
*.md
Dockerfile
compose*.yaml
vendor/
tmp/
```

## Почему именно так

### scratch vs Alpine vs distroless
| Образ | Размер | Shell | Debug | Когда |
|-------|--------|-------|-------|-------|
| `scratch` | 0 MB | нет | невозможен | Максимальная безопасность, минимум размера |
| `distroless` | ~20 MB | нет | ограничен | Нужны ca-certs, timezone без ручного копирования |
| `alpine` | ~7 MB | есть | полный | Нужен shell для entrypoint-скриптов |

### CGO_ENABLED=0
По умолчанию Go может использовать C-библиотеки (CGO) для DNS, сетевых вызовов. `CGO_ENABLED=0` заставляет Go использовать чистую Go-реализацию. Без этого бинарник не запустится на `scratch` (нет libc).

### ldflags="-s -w"
- `-s` — убирает таблицу символов
- `-w` — убирает DWARF debug info

Бинарник 20MB → 14MB. В production отладочная информация не нужна.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| CGO_ENABLED не выключен | `exec format error` на scratch | `CGO_ENABLED=0` |
| Нет ca-certificates | `x509: certificate signed by unknown authority` | Скопировать сертификаты из builder |
| Нет timezone | `time.LoadLocation` возвращает ошибку | Скопировать `/usr/share/zoneinfo` или использовать distroless |
| GOARCH не совпадает | Бинарник не запускается | Указать `GOARCH=amd64` (или `arm64` для ARM) |

## Варианты

- С Nginx: [[docker/how-to/recipes/nginx-reverse-proxy]]
- Private modules: добавить `GOPRIVATE=github.com/myorg/*` + `.netrc` для аутентификации
- С CGO (SQLite, etc): использовать `alpine` вместо `scratch`, добавить `build-base` в builder