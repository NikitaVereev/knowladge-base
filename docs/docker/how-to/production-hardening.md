---
title: "Production Hardening"
type: how-to
tags: [docker, security, hardening, production, non-root, read-only, capabilities, healthcheck]
sources:
  docs: "https://docs.docker.com/engine/security/"
  book: "Docker Deep Dive — Nigel Poulton, Ch.15"
related:
  - "[[docker/explanation/security]]"
  - "[[docker/reference/security-checklist]]"
  - "[[docker/how-to/manage-resources]]"
  - "[[docker/how-to/optimize-builds]]"
---

# Production Hardening

> **TL;DR:** Как превратить рабочий контейнер в production-ready: non-root, read-only,
> минимальные привилегии, лимиты ресурсов, healthcheck.
> Каждый пункт — конкретный код, который можно скопировать.

## Предварительные условия

- Рабочий Dockerfile (см. [[docker/tutorials/02-building-images]])
- Полный чек-лист: [[docker/reference/security-checklist]]
- Теория: [[docker/explanation/security]]

## 1. Non-root пользователь

По умолчанию процесс в контейнере запускается от `root`. Если злоумышленник эксплуатирует уязвимость приложения и совершит container breakout — он окажется на хосте с правами root.

### Dockerfile

```dockerfile
FROM node:20-alpine

# Создаём группу и пользователя с фиксированным UID/GID
# Фиксированный UID нужен для предсказуемого поведения с volumes
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .
RUN npm ci --only=production

# Переключаемся на non-root ПЕРЕД CMD
USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

### Проверка

```bash
# Смотрим от какого пользователя работает процесс
docker exec <container> whoami
# Ожидаем: appuser

docker exec <container> id
# Ожидаем: uid=1001(appuser) gid=1001(appgroup)

# Попытка записи в системные директории — должна провалиться
docker exec <container> touch /etc/test
# Ожидаем: touch: /etc/test: Permission denied
```

### Для Alpine: почему не `USER node`

В образах `node:alpine` уже есть пользователь `node` (UID 1000), и его можно использовать:

```dockerfile
USER node
```

Но у этого подхода минус — UID 1000 часто совпадает с первым пользователем на хосте. Для production лучше создать своего с явным UID.

## 2. Read-only файловая система

Запрещаем запись в корневую ФС контейнера. Приложение может писать только в явно разрешённые места (volumes, tmpfs).

### docker run

```bash
docker run -d \
  --name app \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /app/.next/cache:rw,noexec,nosuid,size=200m \
  -v app-data:/app/data \
  myapp:latest
```

- `--read-only` — корневая ФС монтируется как read-only
- `--tmpfs /tmp` — временная директория в RAM, исчезает при остановке
- `-v app-data:/app/data` — персистентное хранилище для данных

### compose.yaml

```yaml
services:
  app:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp:size=100m
      - /app/.next/cache:size=200m
    volumes:
      - app-data:/app/data
```

### Типичные проблемы с read-only

| Приложение | Куда пишет | Решение |
|------------|-----------|---------|
| Node.js | `/tmp` | `--tmpfs /tmp` |
| Next.js | `.next/cache` | `--tmpfs /app/.next/cache` |
| Nginx | `/var/cache/nginx`, `/var/run` | `--tmpfs /var/cache/nginx --tmpfs /var/run` |
| Python | `__pycache__` | `--tmpfs /app/__pycache__` или `PYTHONDONTWRITEBYTECODE=1` |

## 3. Drop capabilities

Linux capabilities — это дробление прав root на мелкие части. Docker по умолчанию даёт контейнеру ~14 capabilities. Большинству приложений не нужна ни одна.

### docker run

```bash
docker run -d \
  --name app \
  --cap-drop=ALL \
  myapp:latest
```

Если приложению нужно что-то конкретное — добавляем точечно:

```bash
# Веб-сервер на порту < 1024 (например, Nginx на 80)
docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx:alpine

# Приложение, которому нужно менять владельца файлов
docker run -d \
  --cap-drop=ALL \
  --cap-add=CHOWN \
  --cap-add=SETUID \
  --cap-add=SETGID \
  myapp:latest
```

### compose.yaml

```yaml
services:
  app:
    image: myapp:latest
    cap_drop:
      - ALL
    # cap_add:
    #   - NET_BIND_SERVICE    # раскомментировать если нужен порт < 1024
```

### Проверка

```bash
# Смотрим текущие capabilities контейнера
docker exec <container> cat /proc/1/status | grep Cap

# Или через docker inspect
docker inspect <container> --format '{{.HostConfig.CapDrop}}'
# Ожидаем: [ALL]
```

## 4. No new privileges

Запрещаем процессам внутри контейнера повышать привилегии через SUID/SGID бинарники.

### docker run

```bash
docker run -d \
  --security-opt=no-new-privileges:true \
  myapp:latest
```

### compose.yaml

```yaml
services:
  app:
    image: myapp:latest
    security_opt:
      - no-new-privileges:true
```

### Глобально через daemon.json

Чтобы применить ко всем контейнерам на хосте:

```json
{
  "no-new-privileges": true
}
```

## 5. Resource limits

Без лимитов один контейнер может съесть всю RAM на хосте и вызвать OOM для всех остальных процессов.

### compose.yaml (рекомендуемый способ)

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1.0'        # максимум 1 ядро
          memory: 512M        # максимум 512MB RAM
        reservations:
          cpus: '0.25'       # гарантированные 0.25 ядра
          memory: 128M        # гарантированные 128MB RAM
```

### docker run

```bash
docker run -d \
  --name app \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --pids-limit=100 \
  myapp:latest
```

- `--memory=512m` — жёсткий лимит RAM
- `--memory-swap=512m` — то же значение = swap отключён (не даём контейнеру использовать swap)
- `--cpus=1.0` — лимит CPU
- `--pids-limit=100` — максимум процессов (защита от fork-бомб)

### Проверка

```bash
# Текущее потребление в реальном времени
docker stats <container> --no-stream

# Лимиты
docker inspect <container> --format '{{.HostConfig.Memory}}'
# 536870912 (это 512MB в байтах)
```

## 6. Healthcheck

Без healthcheck Docker не знает, работает ли приложение внутри контейнера. Контейнер может быть `running`, но приложение — зависло.

### Dockerfile

```dockerfile
HEALTHCHECK \
  --interval=30s \
  --timeout=5s \
  --start-period=10s \
  --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

- `--interval=30s` — проверять каждые 30 секунд
- `--timeout=5s` — если ответ не пришёл за 5 секунд — считать неудачей
- `--start-period=10s` — дать приложению 10 секунд на старт
- `--retries=3` — 3 неудачи подряд → статус `unhealthy`

### compose.yaml

```yaml
services:
  app:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
```

### Варианты проверок

```dockerfile
# HTTP endpoint (Node.js, Python, Go)
HEALTHCHECK CMD wget --spider http://localhost:3000/health || exit 1

# TCP порт (для сервисов без HTTP, например Redis)
HEALTHCHECK CMD redis-cli ping || exit 1

# PostgreSQL
HEALTHCHECK CMD pg_isready -U postgres || exit 1

# Файл-маркер (если нет сетевого доступа)
HEALTHCHECK CMD test -f /tmp/healthy || exit 1
```

### Проверка

```bash
# Статус healthcheck
docker inspect <container> --format '{{.State.Health.Status}}'
# Ожидаем: healthy

# Последние результаты проверок
docker inspect <container> --format '{{json .State.Health}}' | python3 -m json.tool
```

## 7. Restart policy

Что делать, если контейнер упал.

### compose.yaml

```yaml
services:
  app:
    image: myapp:latest
    restart: unless-stopped    # перезапускать всегда, кроме явной остановки

  worker:
    image: myapp-worker:latest
    restart: on-failure        # перезапускать только при ненулевом exit code
    deploy:
      restart_policy:
        max_attempts: 5        # максимум 5 попыток
        delay: 5s              # пауза между попытками
```

| Policy | Поведение |
|--------|----------|
| `no` | Не перезапускать (по умолчанию) |
| `on-failure` | Только при ненулевом exit code |
| `always` | Всегда, включая после `docker stop` + рестарт Docker |
| `unless-stopped` | Как `always`, но не перезапускает после явного `docker stop` |

## 8. Logging

Без настройки логов контейнер может заполнить весь диск хоста.

### daemon.json (глобально для всех контейнеров)

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### compose.yaml (для конкретного сервиса)

```yaml
services:
  app:
    image: myapp:latest
    logging:
      driver: json-file
      options:
        max-size: "10m"     # максимум 10MB на файл
        max-file: "3"       # максимум 3 файла (ротация)
```

## Собираем всё вместе

Итоговый production-ready `compose.yaml`:

```yaml
services:
  app:
    image: myapp:latest
    read_only: true
    user: "1001:1001"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp:size=100m
    ports:
      - "127.0.0.1:3000:3000"    # только localhost
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `--read-only` без tmpfs | Приложение падает: `EROFS: read-only file system` | Добавить `--tmpfs` для директорий, куда приложение пишет |
| `--cap-drop=ALL` для Nginx на порту 80 | `bind() to 0.0.0.0:80 failed: Permission denied` | Добавить `--cap-add=NET_BIND_SERVICE` или слушать порт > 1024 |
| `USER` в Dockerfile перед `RUN npm ci` | `EACCES: permission denied` при установке пакетов | `USER` ставить после всех `RUN` с установкой |
| Нет `--memory-swap` при `--memory` | Контейнер использует swap и «тормозит» вместо падения | `--memory-swap=512m` (равно `--memory`) — отключает swap |
| Healthcheck с `curl` в Alpine | `curl: not found` — Alpine не содержит curl | Использовать `wget --spider` (wget есть в Alpine) |
| Нет `--pids-limit` | Fork-бомба в контейнере убивает хост | `--pids-limit=100` (или адекватное число для приложения) |

## Связанные материалы

- Чек-лист: [[docker/reference/security-checklist]] — что проверить перед деплоем
- Теория: [[docker/explanation/security]] — capabilities, seccomp, AppArmor
- Ресурсы: [[docker/how-to/manage-resources]] — подробнее про CPU/RAM лимиты

