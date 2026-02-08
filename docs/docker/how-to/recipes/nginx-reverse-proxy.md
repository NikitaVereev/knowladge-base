---
title: "Docker: Nginx Reverse Proxy"
type: recipe
tags: [docker, recipe, nginx, reverse-proxy, ssl, certbot, letsencrypt, production]
sources:
  docs: "https://nginx.org/en/docs/"
  example: "https://github.com/docker/awesome-compose/tree/master/nginx-golang"
related:
  - "[[docker/explanation/networking]]"
  - "[[docker/how-to/recipes/fullstack]]"
  - "[[docker/how-to/recipes/nextjs]]"
---

# Nginx Reverse Proxy в Docker

> Nginx перед приложением: SSL-терминация, статика, сжатие, rate-limiting.
> Production-ready с Let's Encrypt через Certbot.

## Быстрый старт (без SSL)

```bash
docker compose up -d
# → http://localhost → проксирует на app:3000
```

## nginx.conf — базовый reverse proxy

```nginx
upstream app {
    server app:3000;       # имя сервиса в Docker Compose
    # keepalive снижает накладные расходы на TCP handshake
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;

    # Логи
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log warn;

    # Сжатие
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;

    # Ограничение размера тела запроса (загрузка файлов)
    client_max_body_size 10m;

    # Проксирование на приложение
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;

        # WebSocket поддержка (нужна для HMR, Socket.io и т.д.)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Передаём реальный IP клиента
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Таймауты
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }

    # Статика — отдаём напрямую (не проксируем)
    location /static/ {
        alias /var/www/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Health-check endpoint для Docker / балансировщика
    location /nginx-health {
        access_log off;
        return 200 "ok";
    }
}
```

## nginx.conf — production с SSL (Let's Encrypt)

```nginx
upstream app {
    server app:3000;
    keepalive 32;
}

# Редирект HTTP → HTTPS
server {
    listen 80;
    server_name example.com;

    # Certbot challenge (нужен для получения/обновления сертификата)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    # SSL-сертификаты (Let's Encrypt)
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Современные SSL-настройки
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # HSTS — браузер будет принудительно использовать HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Безопасность
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    # Сжатие
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;

    client_max_body_size 10m;

    location / {
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

## compose.yaml — без SSL (development / за CloudFlare)

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost/nginx-health"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  app:
    build: .
    expose:
      - "3000"                 # НЕ публикуем на хост — только для nginx
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

## compose.yaml — production с SSL

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-ssl.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot-www:/var/www/certbot:ro           # challenge файлы
      - certbot-certs:/etc/letsencrypt:ro         # сертификаты
    depends_on:
      app:
        condition: service_healthy
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE       # разрешить bind на порт 80/443
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

  certbot:
    image: certbot/certbot
    volumes:
      - certbot-www:/var/www/certbot
      - certbot-certs:/etc/letsencrypt
    # Первичное получение сертификата:
    # docker compose run certbot certonly --webroot -w /var/www/certbot -d example.com
    #
    # Обновление (добавить в crontab хоста):
    # 0 0 1 * * cd /path/to/project && docker compose run certbot renew && docker compose exec nginx nginx -s reload
    entrypoint: echo "Certbot ready. Run manually."

  app:
    build: .
    expose:
      - "3000"
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    restart: unless-stopped

volumes:
  certbot-www:
  certbot-certs:
```

## Получение SSL-сертификата

```bash
# 1. Запустить nginx без SSL (только HTTP)
# Временно заменить nginx.conf на версию без SSL (только порт 80 + certbot location)
docker compose up -d nginx

# 2. Получить сертификат
docker compose run --rm certbot certonly \
  --webroot \
  -w /var/www/certbot \
  -d example.com \
  --email you@example.com \
  --agree-tos \
  --no-eff-email

# 3. Переключить на SSL-конфиг и перезапустить
docker compose up -d
```

## Почему именно так

### Nginx в отдельном контейнере
Один контейнер = один процесс. Nginx занимается SSL, сжатием и статикой. Приложение — бизнес-логикой. Можно обновлять их независимо.

### expose вместо ports для app
`expose: "3000"` делает порт доступным внутри Docker-сети, но **не** на хосте. Весь внешний трафик идёт через Nginx.

### keepalive 32
Без keepalive Nginx открывает новое TCP-соединение к app на каждый запрос. keepalive держит пул соединений открытыми.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Нет `proxy_set_header Host` | Приложение не знает свой домен | Добавить все `proxy_set_header` |
| Нет `Upgrade` / `Connection` | WebSocket не работает (HMR, Socket.io) | Добавить два заголовка для WebSocket |
| `502 Bad Gateway` | Nginx не может достучаться до app | Проверить: app запущен? Имя сервиса верное? Сеть общая? |
| SSL-сертификат не обновляется | Сертификат истёк через 90 дней | Добавить cron для `certbot renew` |
| Nginx падает при старте без сертификата | `cannot load certificate` | Сначала получить сертификат с HTTP-only конфигом |

## Варианты

- С Next.js: [[docker/how-to/recipes/nextjs]] + этот рецепт
- Полный стек: [[docker/how-to/recipes/fullstack]]
- Вместо Certbot + Nginx: [Traefik](https://doc.traefik.io/traefik/) — автоматический SSL без конфигов
