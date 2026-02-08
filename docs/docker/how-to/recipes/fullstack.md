---
title: "Docker: Fullstack"
type: recipe
tags: [docker, recipe, fullstack, compose, nginx, postgres, redis, production, multi-service]
sources:
  docs: "https://docs.docker.com/compose/gettingstarted/"
  example: "https://github.com/docker/awesome-compose"
related:
  - "[[docker/explanation/compose-model]]"
  - "[[docker/explanation/networking]]"
  - "[[docker/how-to/recipes/nginx-reverse-proxy]]"
  - "[[docker/how-to/recipes/postgres]]"
  - "[[docker/how-to/recipes/redis]]"
  - "[[docker/how-to/production-hardening]]"
---

# Fullstack-приложение в Docker

> Готовый compose.yaml для полного стека: App + PostgreSQL + Redis + Nginx.
> Два варианта: development (hot reload) и production (hardened).
> Приложение — любое (Next.js, Node, Python, Go). Замени `app` на свой Dockerfile.

## Быстрый старт

```bash
# Скопируй compose.yaml, создай .env, запусти
cp .env.example .env
docker compose up -d
# → http://localhost
```

## Архитектура

```
                    ┌─────────────┐
        :80/:443    │    Nginx    │   SSL-терминация, статика, сжатие
   ─────────────────│ reverse     │
        Internet    │ proxy       │
                    └──────┬──────┘
                           │ :3000 (Docker network)
                    ┌──────┴──────┐
                    │     App     │   Бизнес-логика
                    │ (Next.js /  │
                    │  Node / Go) │
                    └──┬──────┬───┘
              :5432    │      │ :6379
          ┌────────────┘      └────────────┐
   ┌──────┴──────┐              ┌──────────┴──────┐
   │ PostgreSQL  │              │     Redis       │
   │ (данные)    │              │ (кэш/сессии)    │
   └─────────────┘              └─────────────────┘
```

Nginx и App — в `frontend-net`. App, PostgreSQL, Redis — в `backend-net`.
PostgreSQL и Redis **не доступны** из интернета и друг другу не видны напрямую.

## .env.example

```bash
# Приложение
NODE_ENV=production
APP_PORT=3000

# PostgreSQL
POSTGRES_DB=myapp
POSTGRES_USER=myapp
POSTGRES_PASSWORD=change_me_in_production

# Redis
REDIS_PASSWORD=change_me_in_production

# Nginx
DOMAIN=example.com
```

## compose.yaml — development

```yaml
services:
  # ─── Приложение ───
  app:
    build:
      context: .
      target: deps                 # стадия зависимостей (без production-сборки)
    command: npm run dev           # или: uvicorn app.main:app --reload
    ports:
      - "3000:3000"               # прямой доступ без Nginx в dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  # ─── PostgreSQL ───
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"               # доступ с хоста для DBeaver / pgAdmin
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ─── Redis ───
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"               # доступ с хоста для redis-cli
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  db-data:
  redis-data:
```

## compose.yaml — production

```yaml
services:
  # ─── Nginx (точка входа) ───
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot-www:/var/www/certbot:ro
      - certbot-certs:/etc/letsencrypt:ro
    networks:
      - frontend-net
    depends_on:
      app:
        condition: service_healthy
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost/nginx-health"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  # ─── Приложение ───
  app:
    build:
      context: .
      target: runner                # production-стадия
    expose:
      - "3000"                      # доступно только для nginx, не на хосте
    read_only: true
    tmpfs:
      - /tmp:size=100m
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    networks:
      - frontend-net                # nginx → app
      - backend-net                 # app → db, redis
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
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

  # ─── PostgreSQL ───
  db:
    image: postgres:16-alpine
    # НЕ публикуем порт — доступ только через backend-net
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend-net
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          memory: 512M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      start_period: 20s
      retries: 5
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # ─── Redis ───
  redis:
    image: redis:7-alpine
    command: >
      redis-server
        --maxmemory 128mb
        --maxmemory-policy allkeys-lru
        --requirepass ${REDIS_PASSWORD}
        --save ""
        --appendonly no
    networks:
      - backend-net
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 200M
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ─── Certbot (SSL) ───
  certbot:
    image: certbot/certbot
    volumes:
      - certbot-www:/var/www/certbot
      - certbot-certs:/etc/letsencrypt
    entrypoint: echo "Run manually: docker compose run certbot certonly --webroot -w /var/www/certbot -d ${DOMAIN}"

volumes:
  db-data:
  certbot-www:
  certbot-certs:

networks:
  frontend-net:
    # nginx ↔ app
  backend-net:
    # app ↔ db, redis
    # nginx НЕ подключён к backend-net → не видит db и redis
```

## nginx.conf

```nginx
upstream app {
    server app:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name ${DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        # Для начала — без SSL. После получения сертификата → redirect на HTTPS
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /nginx-health {
        access_log off;
        return 200 "ok";
    }
}
```

## Порядок запуска

```bash
# 1. Создать .env из примера
cp .env.example .env
# Заменить пароли на реальные!

# 2. Запустить всё
docker compose up -d

# 3. Проверить статус
docker compose ps
# Все сервисы должны быть healthy

# 4. Посмотреть логи если что-то не работает
docker compose logs app     # логи приложения
docker compose logs db      # логи PostgreSQL
docker compose logs nginx   # логи Nginx

# 5. SSL (когда DNS настроен на сервер)
docker compose run --rm certbot certonly \
  --webroot -w /var/www/certbot \
  -d example.com \
  --email you@example.com --agree-tos --no-eff-email
# После получения сертификата — обновить nginx.conf на HTTPS
```

## Почему именно так

### Две сети вместо одной
`frontend-net` (nginx ↔ app) + `backend-net` (app ↔ db, redis). Nginx не видит базу данных. Если Nginx скомпрометирован — злоумышленник не доберётся до БД напрямую.

### expose вместо ports для app
`expose: "3000"` делает порт видимым внутри Docker-сети, но **не** на хосте. Весь внешний трафик идёт через Nginx. На хосте открыты только порты 80 и 443.

### depends_on + condition: service_healthy
Без healthcheck приложение может стартовать раньше, чем PostgreSQL будет готов принимать соединения. `condition: service_healthy` ждёт, пока `pg_isready` вернёт OK.

### Redis как кэш (save "" + appendonly no)
Для кэша данные не нужно сохранять на диск. Если Redis перезапустится — кэш прогреется заново. Это экономит I/O и упрощает конфигурацию.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Пароли в compose.yaml | Видны в `docker inspect` и git | Использовать `.env` файл (в `.gitignore`) или Docker secrets |
| Порты БД опубликованы | PostgreSQL доступен из интернета | Убрать `ports:` для db и redis |
| Нет healthcheck на db | App падает с `ECONNREFUSED` | Добавить `healthcheck` + `depends_on: condition` |
| Одна сеть на все сервисы | Nginx видит PostgreSQL | Две сети: frontend-net + backend-net |
| Нет ротации логов | Диск переполняется через месяц | `logging: max-size: 10m` |
| `.env` в git | Пароли утекли | Добавить `.env` в `.gitignore`, создать `.env.example` |

## Масштабирование

```bash
# Запустить 3 инстанса приложения за Nginx
docker compose up -d --scale app=3
# Nginx автоматически балансирует между ними (DNS round-robin)
```

## Варианты

- С мониторингом: добавить [[docker/how-to/recipes/monitoring]]
- Next.js как app: [[docker/how-to/recipes/nextjs]]
- Django как app: [[docker/how-to/recipes/python-django]]
- Без Nginx (за CloudFlare): убрать сервис nginx, опубликовать app напрямую

````
