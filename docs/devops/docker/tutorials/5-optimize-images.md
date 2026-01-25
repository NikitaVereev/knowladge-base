---
title: "5 Оптимизация - Multi-stage Build"
description: "Как уменьшить размер Docker-образа в 20 раз (с 1 ГБ до 50 МБ) с помощью многоэтапной сборки."
---

Одна из главных проблем Docker — огромные образы. Например, стандартный образ с Go или Node.js весит под 800 МБ, потому что содержит компиляторы, заголовочные файлы и кучу системных утилит.

Для продакшена вам нужен только скомпилированный бинарник. Решение — **Multi-stage Build**.

## Пример: Golang
Задача: Скомпилировать Go-приложение и упаковать его в минимальный контейнер.

### Плохой Dockerfile (Single-stage)
```dockerfile
FROM golang:1.24
WORKDIR /app
COPY . .
RUN go build -o myapp
CMD ["./myapp"]
```
*   **Результат:** ~900 MB (так как внутри весь Go SDK, исходники и кэши).

### Хороший Dockerfile (Multi-stage)
Мы делим процесс на два этапа: "Сборка" и "Финальный образ".

```dockerfile
# --- Этап 1: Builder ---
# Берем тяжелый образ с полным Go SDK
FROM golang:1.24 AS builder

WORKDIR /app

# Оптимизация кэша: сначала качаем зависимости
COPY go.mod go.sum ./
RUN go mod download

# Копируем код и собираем
COPY . .
# CGO_ENABLED=0 создает статический бинарник без внешних зависимостей
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .


# --- Этап 2: Runtime ---
# Берем пустой базовый образ (или alpine, если нужны утилиты)
FROM alpine:latest

WORKDIR /root/

# Копируем ИЗ первого этапа ТОЛЬКО готовый файл
COPY --from=builder /app/myapp .

# Запускаем
CMD ["./myapp"]
```

*   **Результат:** ~15 MB (Alpine + 5MB бинарник).
*   **Выигрыш:** В 60 раз меньше места! В финальном образе нет исходного кода (безопаснее) и компилятора.

## Пример: Node.js (Frontend / React)
Для статических сайтов (React/Vue) мы собираем проект Node.js-ом, а раздаем Nginx-ом.

```dockerfile
# Stage 1: Build
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build  # Создает папку /app/dist

# Stage 2: Serve
FROM nginx:alpine
# Копируем только статику в папку Nginx
COPY --from=builder /app/dist /usr/share/nginx/html
# Nginx запускается сам по умолчанию
```

## Советы по оптимизации
1.  **Alpine vs Distroless vs Scratch:**
    *   `alpine`: Маленький (5MB), есть шелл `sh` (удобно отлаживать).
    *   `gcr.io/distroless/static`: Чуть больше, но **нет шелла** (супер безопасно).
    *   `scratch`: Абсолютно пустой образ (0 байт). Работает только для статически скомпилированных бинарников (Go, Rust).
2.  **Docker Slim:** Если лень писать мультистейдж руками, попробуйте утилиту `docker-slim`, которая автоматически выкидывает мусор из готового образа.
