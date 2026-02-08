---
title: "Создание и использование ролей"
type: how-to
tags: [ansible, roles, galaxy, ansible-galaxy, init, defaults, meta, dependencies]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html"
related:
  - "[[ansible/explanation/roles-and-collections]]"
  - "[[ansible/how-to/test-with-molecule]]"
  - "[[ansible/reference/project-structure]]"
---

# Создание и использование ролей

> **TL;DR:** `ansible-galaxy init my_role` — scaffold. Настраиваемые параметры — в `defaults/`.
> Константы — в `vars/`. Зависимости — в `meta/`. Galaxy — хаб готовых ролей.

## Создание роли

```bash
# Scaffold с помощью Galaxy
ansible-galaxy init roles/nginx

# Или вручную — только нужные директории
mkdir -p roles/nginx/{tasks,handlers,defaults,templates,files}
```

### Минимальная структура

```
roles/nginx/
├── tasks/
│   └── main.yml          # точка входа
├── handlers/
│   └── main.yml          # restart/reload
├── defaults/
│   └── main.yml          # настраиваемые параметры (низкий приоритет)
├── templates/
│   └── nginx.conf.j2     # шаблоны конфигов
└── README.md
```

### Пример: роль nginx

**defaults/main.yml** — параметры, которые пользователь роли может переопределить:

```yaml
---
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_name: "{{ ansible_fqdn }}"
nginx_root: /var/www/html
nginx_enable_ssl: false
```

**tasks/main.yml**:

```yaml
---
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: yes
  tags: [install]

- name: Create web root
  ansible.builtin.file:
    path: "{{ nginx_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'

- name: Deploy config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: "nginx -t -c %s"
  notify: Reload Nginx
  tags: [config]

- name: Enable and start
  ansible.builtin.systemd:
    name: nginx
    state: started
    enabled: yes
```

**handlers/main.yml**:

```yaml
---
- name: Reload Nginx
  ansible.builtin.systemd:
    name: nginx
    state: reloaded

- name: Restart Nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
```

## Использование в Playbook

```yaml
- hosts: webservers
  become: yes
  roles:
    # Простой вызов (defaults)
    - nginx

    # С параметрами
    - role: nginx
      vars:
        nginx_port: 8080
        nginx_enable_ssl: true

    # Условный вызов
    - role: monitoring
      when: enable_monitoring | default(false) | bool
```

## Зависимости (meta/main.yml)

```yaml
---
dependencies:
  - role: common             # выполнится перед nginx
  - role: certbot
    when: nginx_enable_ssl
    vars:
      certbot_domain: "{{ nginx_server_name }}"
```

## Ansible Galaxy

```bash
# Поиск
ansible-galaxy search nginx
ansible-galaxy info geerlingguy.nginx

# Установка одной роли
ansible-galaxy install geerlingguy.nginx

# Установка из requirements.yml
ansible-galaxy install -r requirements.yml
```

**requirements.yml**:

```yaml
roles:
  - name: geerlingguy.docker
    version: "7.1.0"
  - name: geerlingguy.nginx
  - src: https://github.com/myorg/private-role.git
    name: private_role
    version: main

collections:
  - name: community.docker
    version: ">=3.0.0"
  - name: community.postgresql
```

```bash
# Установить всё из requirements
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Параметры в `vars/` вместо `defaults/` | Невозможно переопределить из playbook | Настраиваемые параметры → `defaults/`. `vars/` — только для констант |
| Роль без README | Непонятно какие параметры принимает | Всегда документировать defaults в README |
| Galaxy-роль без версии | `requirements.yml` ставит latest → сломалось при обновлении | Фиксировать `version:` |
| Все задачи в одном tasks/main.yml | 500-строчный файл | Разбить: `tasks/install.yml`, `tasks/config.yml` + `include_tasks` |