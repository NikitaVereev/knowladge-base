---
title: "4 Best Practices"
description: "Золотые правила работы с Docker: безопасность, оптимизация Dockerfile (кэширование, multistage) и правильная эксплуатация."
---

В этом разделе собраны стандарты индустрии, которые отличают "работает на моей машине" от "Production Ready" решения.

## 1. Безопасность (Security)

### Run as Non-Root
По умолчанию контейнер запускается от `root`. Если злоумышленник "сбежит" из контейнера (Container Breakout), он получит root-права на хосте.
**Решение:** Всегда создавайте и используйте непривилегированного пользователя.
```dockerfile
# Создаем группу и юзера
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
# ... установка зависимостей ...
USER appuser
CMD ["node", "app.js"]
```

### Минимальные образы (Distroless / Alpine)
Чем меньше в образе утилит, тем меньше векторов атаки.
*   **Alpine:** Очень маленький (~5MB), но использует `musl libc` (иногда проблемы совместимости).
*   **Distroless (Google):** В образе вообще нет шелла (`sh`/`bash`)! Только ваше приложение и runtime. Идеально для продакшена.

### Управление секретами
Никогда не используйте `ENV` для передачи паролей в Dockerfile или `docker-compose.yml` (если репозиторий публичный).
*   **Плохо:** `ENV DB_PASSWORD=secret` (видно в `docker history`).
*   **Хорошо:** Использовать Docker Secrets (в Swarm) или монтировать файлы секретов на этапе запуска.

## 2. Оптимизация сборки (Dockerfile)

### Правильный порядок слоев (Caching)
Docker кэширует слои. Если слой изменился, все последующие слои пересобираются.
*   **Правило:** Сначала копируем то, что меняется редко (зависимости), потом то, что часто (код).

```dockerfile
WORKDIR /app
# 1. Сначала копируем только файлы зависимостей
COPY package.json package-lock.json ./
# 2. Ставим пакеты (этот слой закэшируется надолго)
RUN npm ci --only=production
# 3. И только потом копируем код
COPY . .
CMD ["node", "index.js"]
```

### .dockerignore
Работает как `.gitignore`. Обязательно добавьте туда мусор, чтобы не отправлять гигабайты `node_modules` или `.git` демону сборки.
```text
.git
node_modules
Dockerfile
.env
```

### Multi-stage Builds
Позволяют отделить инструменты сборки (компиляторы, исходники) от финального артефакта.
```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Run (Чистый образ без компилятора Go)
FROM alpine:latest
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
```

## 3. Runtime (Эксплуатация)

### One Process per Container
Один контейнер = один процесс (PID 1). Не пытайтесь запихнуть Nginx + PHP + MySQL в один Dockerfile через supervisor.
*   *Почему:* Сложно масштабировать, сложно читать логи, сложно перезапускать по отдельности. Используйте Docker Compose для связки сервисов.

### Логирование в STDOUT/STDERR
Приложение не должно писать логи в файлы (`/var/log/myapp.log`). Оно должно писать в стандартный вывод.
*   *Почему:* Docker сам перехватывает эти потоки (`docker logs`) и позволяет перенаправлять их в ELK/Graylog/CloudWatch без изменения кода приложения.

### Healthchecks
Docker должен знать, живое ли приложение на самом деле (а не просто "процесс запущен").
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/health || exit 1
```
Если healthcheck провалится, Docker пометит контейнер `unhealthy`, и оркестратор (Swarm/K8s) сможет его перезапустить.
