---
title: "Работа с шаблонами (Jinja2)"
type: how-to
tags: [ansible, templates, jinja2, filters, nginx, configuration, envsubst]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html"
related:
  - "[[ansible/reference/jinja2-filters]]"
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/how-to/write-playbooks]]"
---

# Работа с шаблонами (Jinja2)

> **TL;DR:** `template` модуль + файл `.j2` = динамическая генерация конфигов.
> `{{ переменная }}` — подстановка, `{% if %}` — условия, `{% for %}` — циклы.
> Шаблон обрабатывается на Control Node, результат копируется на сервер.

Модуль `template` берёт файл-шаблон (`.j2`), обрабатывает его Jinja2, и копирует результат на сервер.

## Создание шаблона

Файл `templates/nginx.conf.j2`:

```nginx
# Managed by Ansible — DO NOT EDIT MANUALLY
server {
    listen {{ nginx_port | default(80) }};
    server_name {{ ansible_fqdn }};

    root {{ web_root | default('/var/www/html') }};
    index index.html;

    # Условный блок
    {% if enable_ssl | default(false) %}
    listen 443 ssl;
    ssl_certificate     /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;
    {% endif %}

    # Цикл по upstream-серверам
    {% for backend in app_backends | default([]) %}
    location {{ backend.path }} {
        proxy_pass {{ backend.url }};
        proxy_set_header Host $host;
    }
    {% endfor %}

    # Комментарий Jinja2 (не попадёт в файл)
    {# Этот блок для будущего расширения #}
}
```

## Использование в Playbook

```yaml
- name: Настроить Nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/sites-available/default
    owner: root
    group: root
    mode: '0644'
    validate: "nginx -t -c %s"       # валидация перед применением
    backup: yes                       # сохранить предыдущую версию
  notify: Reload Nginx
```

## Переменные для шаблона

```yaml
# group_vars/webservers.yml
nginx_port: 8080
enable_ssl: true
app_backends:
  - path: /api
    url: http://localhost:3000
  - path: /admin
    url: http://localhost:8000
```

## Синтаксис Jinja2

### Подстановки

```jinja2
{{ variable }}                         {# простая подстановка #}
{{ variable | default('fallback') }}   {# значение по умолчанию #}
{{ variable | upper }}                 {# фильтр: в верхний регистр #}
{{ list_var | join(', ') }}            {# склеить список #}
{{ password | hash('sha512') }}        {# хеш пароля #}
```

### Условия

```jinja2
{% if env == 'production' %}
worker_processes auto;
{% elif env == 'staging' %}
worker_processes 2;
{% else %}
worker_processes 1;
{% endif %}
```

### Циклы

```jinja2
{% for user in users %}
{{ user.name }}:{{ user.uid }}:{{ user.shell | default('/bin/bash') }}
{% endfor %}

{# С индексом #}
{% for item in items %}
server_{{ loop.index }} = {{ item }}
{% endfor %}

{# loop.index  — счётчик с 1 #}
{# loop.index0 — счётчик с 0 #}
{# loop.first  — первая итерация? #}
{# loop.last   — последняя итерация? #}
```

### Управление пробелами

```jinja2
{# По умолчанию Jinja2 оставляет пустые строки вместо блоков #}
{# Используй - для обрезки пробелов #}

{% for server in servers -%}
server {{ server }};
{%- endfor %}
```

## Практические примеры

### systemd unit

```jinja2
# templates/app.service.j2
[Unit]
Description={{ app_name }}
After=network.target

[Service]
User={{ app_user }}
WorkingDirectory={{ app_dir }}
ExecStart={{ app_dir }}/venv/bin/gunicorn \
  --workers {{ ansible_processor_vcpus * 2 + 1 }} \
  --bind 0.0.0.0:{{ app_port }} \
  {{ app_module }}:app
Restart=always
Environment=NODE_ENV={{ env }}

[Install]
WantedBy=multi-user.target
```

### SSH config

```jinja2
# templates/sshd_config.j2
Port {{ ssh_port | default(22) }}
PermitRootLogin {{ 'yes' if allow_root_login | default(false) else 'no' }}
PasswordAuthentication {{ 'yes' if allow_password_auth | default(false) else 'no' }}
MaxAuthTries 3
{% for key in ssh_allowed_keys | default([]) %}
AuthorizedKeysFile {{ key }}
{% endfor %}
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `template` без `notify` | Конфиг обновлён, сервис не перезапущен | Всегда добавлять `notify` для сервисов |
| Нет `validate` | Невалидный конфиг ломает сервис | `validate: "nginx -t -c %s"` перед применением |
| Пробелы и пустые строки | Лишние пустые строки в конфиге | Использовать `-%}` для обрезки whitespace |
| Переменная не определена | `AnsibleUndefinedVariable` | Использовать `\| default('value')` или определить в defaults |