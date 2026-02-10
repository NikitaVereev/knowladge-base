---
title: "01 — Первый Playbook"
type: tutorial
tags: [ansible, tutorial, beginner, playbook, ping, apt, template, handler]
sources:
  docs: "https://docs.ansible.com/ansible/latest/getting_started/index.html"
related:
  - "[[ansible/explanation/architecture]]"
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/how-to/install-and-configure]]"
  - "[[ansible/tutorials/02-server-setup]]"
---

# Tutorial 01 — Первый Playbook

> **Цель:** От нуля до рабочего playbook за 6 шагов. Установить Nginx,
> развернуть страницу, настроить handler. Понять lifecycle: inventory → play → task → handler.

**Время:** ~30 минут
**Требования:** Control Node с Ansible, один сервер (Ubuntu) с SSH-доступом.

## Шаг 1. Структура проекта

```bash
mkdir -p ansible-tutorial && cd ansible-tutorial
```

Создаём 3 файла:

```
ansible-tutorial/
├── ansible.cfg
├── inventory.yml
└── site.yml
```

## Шаг 2. Конфигурация

**ansible.cfg:**

```ini
[defaults]
inventory = ./inventory.yml
host_key_checking = False
stdout_callback = yaml
retry_files_enabled = False

[ssh_connection]
pipelining = True
```

**inventory.yml** (замените IP на ваш сервер):

```yaml
all:
  hosts:
    web:
      ansible_host: 192.168.1.10    # ← ваш IP
      ansible_user: ubuntu           # ← ваш SSH-пользователь
```

## Шаг 3. Проверка подключения

```bash
# Ping — проверка SSH + Python
ansible all -m ping
```

Ожидаемый результат:

```
web | SUCCESS => {
    "ping": "pong"
}
```

Если ошибка — проверьте SSH-ключ и IP.

```bash
# Ad-hoc: какая ОС на сервере?
ansible all -m setup -a "filter=ansible_distribution*"
```

## Шаг 4. Первый Playbook

**site.yml:**

```yaml
---
- name: Настройка веб-сервера
  hosts: all
  become: yes                        # sudo

  vars:
    site_title: "Hello from Ansible!"
    http_port: 80

  tasks:
    # 1. Обновляем кэш пакетов (не чаще раза в час)
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    # 2. Устанавливаем Nginx
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    # 3. Запускаем и включаем автостарт
    - name: Start and enable Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: yes

    # 4. Деплоим нашу страницу
    - name: Deploy index page
      ansible.builtin.copy:
        content: |
          <!DOCTYPE html>
          <html>
          <head><title>{{ site_title }}</title></head>
          <body>
            <h1>{{ site_title }}</h1>
            <p>Server: {{ ansible_hostname }}</p>
            <p>OS: {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
            <p>IP: {{ ansible_default_ipv4.address }}</p>
          </body>
          </html>
        dest: /var/www/html/index.html
        mode: '0644'
      notify: Reload Nginx

    # 5. Проверяем что сайт отвечает
    - name: Verify site is accessible
      ansible.builtin.uri:
        url: "http://localhost:{{ http_port }}"
        status_code: 200
      register: site_check

    - name: Show result
      ansible.builtin.debug:
        msg: "✅ Site is live! Status: {{ site_check.status }}"

  handlers:
    - name: Reload Nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
```

## Шаг 5. Запуск

```bash
# Dry run — посмотреть что будет сделано (без изменений)
ansible-playbook site.yml --check --diff

# Боевой запуск
ansible-playbook site.yml
```

Ожидаемый вывод:

```
TASK [Install Nginx] ****
changed: [web]

TASK [Deploy index page] ****
changed: [web]

RUNNING HANDLER [Reload Nginx] ****
changed: [web]

TASK [Show result] ****
ok: [web] => {
    "msg": "✅ Site is live! Status: 200"
}

PLAY RECAP ****
web  :  ok=7  changed=3  unreachable=0  failed=0
```

Откройте `http://192.168.1.10` в браузере — увидите вашу страницу.

## Шаг 6. Повторный запуск (идемпотентность)

```bash
ansible-playbook site.yml
```

```
PLAY RECAP ****
web  :  ok=6  changed=0  unreachable=0  failed=0
```

`changed=0` — Ansible убедился что всё уже настроено и ничего не менял. Это идемпотентность.

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| inventory | Файл `inventory.yml` с хостом и переменными подключения |
| play | Блок `- name: Настройка веб-сервера` с `hosts` и `become` |
| tasks | Последовательные задачи с модулями (`apt`, `systemd`, `copy`, `uri`) |
| vars | Переменные `site_title` и `http_port` |
| facts | Автоматические `ansible_hostname`, `ansible_distribution` |
| handlers | `Reload Nginx` выполнился только потому что `copy` сделал `changed` |
| идемпотентность | Повторный запуск — `changed=0` |

## Что дальше

→ [[ansible/tutorials/02-server-setup]] — полная настройка сервера (SSH hardening, firewall, пользователи, Docker)

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| SSH-ключ не скопирован | `UNREACHABLE! Permission denied` | `ssh-copy-id ubuntu@192.168.1.10` |
| Python не установлен на сервере | `MODULE FAILURE: /usr/bin/python not found` | `ansible all -m raw -a "apt install -y python3" -b` |
| `become: yes` не указан | `E: Could not open lock file` (нет sudo) | Добавить `become: yes` на уровне play |
| handler не сработал | Nginx не перезагрузился | handler вызывается только если задача вернула `changed` |