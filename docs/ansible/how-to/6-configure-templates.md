---
title: "6 Работа с шаблонами (Jinja2)"
description: "Как динамически генерировать конфигурационные файлы используя переменные и логику."
---

Модуль `template` — это мощнейший инструмент Ansible. Он позволяет брать файл-шаблон (обычно с расширением `.j2`), обрабатывать его движком Jinja2 и копировать результат на сервер.

## 1. Создание шаблона

Создайте файл `templates/nginx.conf.j2`. Вы можете использовать переменные Ansible внутри `{{ ... }}`.

```nginx
server {
    listen {{ nginx_port | default(80) }};
    server_name {{ ansible_fqdn }};

    root /var/www/html;
    index index.html;

    # Условный блок
    {% if enable_ssl %}
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;
    {% endif %}

    # Цикл по списку
    {% for location in nginx_locations %}
    location {{ location.path }} {
        proxy_pass {{ location.target }};
    }
    {% endfor %}
}
```

## 2. Использование в Playbook

```yaml
- name: Configure Nginx
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/sites-available/default
    owner: root
    group: root
    mode: '0644'
  notify: Restart Nginx  # Перезагрузить сервис при изменении конфига
```

## 3. Переменные для шаблона

Их можно задать в `vars`, `group_vars` или инвентаре.

```yaml
vars:
  nginx_port: 8080
  enable_ssl: true
  nginx_locations:
    - path: /api
      target: http://localhost:3000
    - path: /static
      target: http://localhost:80/static
```
