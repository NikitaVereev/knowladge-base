---
title: "Рецепт: Nginx Virtual Host"
type: how-to
tags: [linux, recipe, nginx, virtualhost, ssl, certbot, letsencrypt, reverse-proxy]
related:
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
---

# Рецепт: Nginx Virtual Host

> Nginx: статический сайт, reverse proxy, TLS с Let's Encrypt (certbot).

## Установка

```bash
# Ubuntu/Debian
sudo apt install nginx certbot python3-certbot-nginx

# Arch
sudo pacman -S nginx certbot certbot-nginx

sudo systemctl enable --now nginx
```

## Статический сайт

```bash
# Создать директорию
sudo mkdir -p /var/www/example.com
sudo chown -R $USER:$USER /var/www/example.com
echo "<h1>Hello from example.com</h1>" > /var/www/example.com/index.html
```

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;

    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Логи
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;
}
```

```bash
# Включить сайт
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t                      # проверить конфиг
sudo systemctl reload nginx
```

## Reverse Proxy (Node.js, Python, etc.)

```nginx
# /etc/nginx/sites-available/app.example.com
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## TLS с Let's Encrypt

```bash
# Получить сертификат (автоматически настроит Nginx)
sudo certbot --nginx -d example.com -d www.example.com

# Проверить автообновление
sudo certbot renew --dry-run

# Certbot добавит cron/timer автоматически
systemctl list-timers | grep certbot
```

После certbot конфиг автоматически обновится:

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;

    root /var/www/example.com;
    # ...
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;  # redirect HTTP → HTTPS
}
```

## Arch Linux (без sites-enabled)

В Arch нет `sites-available/sites-enabled`. Используйте include:

```nginx
# /etc/nginx/nginx.conf
http {
    include /etc/nginx/conf.d/*.conf;
}
```

```bash
# Конфиги в /etc/nginx/conf.d/example.com.conf
sudo nginx -t && sudo systemctl reload nginx
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `nginx -t`: syntax error | Проверить точки с запятой и скобки |
| 502 Bad Gateway | Backend (Node/Python) не запущен или неверный порт в proxy_pass |
| certbot: DNS не настроен | A-запись домена должна указывать на IP сервера |
| Забыли symlink в sites-enabled | `ln -s /etc/nginx/sites-available/site /etc/nginx/sites-enabled/` |
| Порт 80 занят | `ss -tlnp | grep :80` — возможно Apache. Остановить его |
