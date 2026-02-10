---
title: "Рецепт: Nginx + Certbot (SSL)"
type: how-to
tags: [ansible, recipe, nginx, certbot, letsencrypt, ssl, reverse-proxy, https]
related:
  - "[[ansible/how-to/recipes/server-bootstrap]]"
  - "[[ansible/how-to/use-templates]]"
  - "[[docker/how-to/recipes/nginx-reverse-proxy]]"
---

# Рецепт: Nginx + Let's Encrypt SSL

> Nginx как reverse proxy с автоматическим SSL-сертификатом через Certbot. HTTP → HTTPS redirect, auto-renewal.

## Playbook

```yaml
---
- name: Setup Nginx with SSL
  hosts: webservers
  become: yes

  vars:
    nginx_domain: app.example.com
    nginx_email: admin@example.com
    nginx_backend_port: 3000
    nginx_ssl_enabled: true

  tasks:
    # === NGINX ===
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Remove default site
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx

    - name: Deploy HTTP config (pre-SSL)
      ansible.builtin.copy:
        content: |
          server {
              listen 80;
              server_name {{ nginx_domain }};

              location /.well-known/acme-challenge/ {
                  root /var/www/certbot;
              }

              location / {
                  {% if nginx_ssl_enabled %}
                  return 301 https://$server_name$request_uri;
                  {% else %}
                  proxy_pass http://127.0.0.1:{{ nginx_backend_port }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
                  {% endif %}
              }
          }
        dest: "/etc/nginx/sites-available/{{ nginx_domain }}"
      notify: Reload Nginx

    - name: Enable site
      ansible.builtin.file:
        src: "/etc/nginx/sites-available/{{ nginx_domain }}"
        dest: "/etc/nginx/sites-enabled/{{ nginx_domain }}"
        state: link
      notify: Reload Nginx

    - name: Validate Nginx config
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Start Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: yes

    # === CERTBOT ===
    - name: Install Certbot
      ansible.builtin.apt:
        name:
          - certbot
          - python3-certbot-nginx
        state: present
      when: nginx_ssl_enabled

    - name: Create webroot directory
      ansible.builtin.file:
        path: /var/www/certbot
        state: directory
        owner: www-data
      when: nginx_ssl_enabled

    - name: Check if certificate exists
      ansible.builtin.stat:
        path: "/etc/letsencrypt/live/{{ nginx_domain }}/fullchain.pem"
      register: cert_file
      when: nginx_ssl_enabled

    - name: Obtain SSL certificate
      ansible.builtin.command: >
        certbot certonly --nginx
        -d {{ nginx_domain }}
        --email {{ nginx_email }}
        --agree-tos
        --non-interactive
      when:
        - nginx_ssl_enabled
        - not cert_file.stat.exists | default(false)

    # === HTTPS CONFIG ===
    - name: Deploy HTTPS config
      ansible.builtin.copy:
        content: |
          # HTTP → HTTPS redirect
          server {
              listen 80;
              server_name {{ nginx_domain }};

              location /.well-known/acme-challenge/ {
                  root /var/www/certbot;
              }

              location / {
                  return 301 https://$server_name$request_uri;
              }
          }

          # HTTPS
          server {
              listen 443 ssl http2;
              server_name {{ nginx_domain }};

              ssl_certificate /etc/letsencrypt/live/{{ nginx_domain }}/fullchain.pem;
              ssl_certificate_key /etc/letsencrypt/live/{{ nginx_domain }}/privkey.pem;

              # SSL settings
              ssl_protocols TLSv1.2 TLSv1.3;
              ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
              ssl_prefer_server_ciphers off;
              ssl_session_cache shared:SSL:10m;

              # Security headers
              add_header X-Frame-Options DENY;
              add_header X-Content-Type-Options nosniff;
              add_header Strict-Transport-Security "max-age=63072000" always;

              location / {
                  proxy_pass http://127.0.0.1:{{ nginx_backend_port }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                  proxy_set_header X-Forwarded-Proto $scheme;
                  proxy_http_version 1.1;
                  proxy_set_header Upgrade $http_upgrade;
                  proxy_set_header Connection "upgrade";
              }
          }
        dest: "/etc/nginx/sites-available/{{ nginx_domain }}"
      notify: Reload Nginx
      when: nginx_ssl_enabled and (cert_file.stat.exists | default(false) or not cert_file.skipped | default(false))

    # === AUTO-RENEWAL ===
    - name: Setup auto-renewal cron
      ansible.builtin.cron:
        name: "Certbot auto-renewal"
        minute: "30"
        hour: "3"
        day: "*/7"
        job: "certbot renew --quiet --post-hook 'systemctl reload nginx'"
        user: root
      when: nginx_ssl_enabled

  handlers:
    - name: Reload Nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
```

## Запуск

```bash
# Без SSL (для тестирования)
ansible-playbook nginx.yml -e "nginx_ssl_enabled=false"

# С SSL (домен должен резолвиться на сервер)
ansible-playbook nginx.yml -e "nginx_domain=myapp.example.com nginx_email=me@example.com"
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Certbot не может получить сертификат | DNS A-запись должна указывать на IP сервера. Порт 80 открыт |
| Nginx не стартует после добавления SSL | Сертификат ещё не получен. Сначала HTTP-конфиг, потом certbot, потом HTTPS-конфиг |
| Сертификат не обновляется | Проверить cron: `crontab -l`. Убедиться что webroot доступен |
| `ERR_TOO_MANY_REDIRECTS` | HTTP→HTTPS redirect loop. Проверить proxy_set_header X-Forwarded-Proto |