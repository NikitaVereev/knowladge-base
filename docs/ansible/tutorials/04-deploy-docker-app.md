---
title: "04 — Деплой Docker-приложения"
type: tutorial
tags: [ansible, tutorial, docker, compose, deploy, rolling, healthcheck, rollback]
sources:
  docs: "https://docs.ansible.com/ansible/latest/collections/community/docker/"
related:
  - "[[ansible/tutorials/03-roles-project]]"
  - "[[ansible/how-to/deploy-strategies]]"
  - "[[docker/how-to/recipes/fullstack]]"
---

# Tutorial 04 — Деплой Docker-приложения

> **Цель:** Автоматический деплой Docker Compose приложения (app + DB + Nginx) через Ansible.
> Rolling update, healthcheck, rollback при ошибке. Практика: template, block/rescue, serial, vault.

**Время:** ~50 минут
**Требования:** Пройден Tutorial 03. Сервер с Docker (из Tutorial 02).

## Шаг 1. Структура проекта

```bash
mkdir -p deploy-app/{roles/app/{tasks,handlers,defaults,templates,files},inventory/group_vars/all}
cd deploy-app
```

```
deploy-app/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all/
│           ├── vars.yml
│           └── vault.yml         # DB пароли
├── roles/
│   └── app/
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── deploy.yml
│       │   └── rollback.yml
│       ├── handlers/main.yml
│       ├── defaults/main.yml
│       └── templates/
│           ├── compose.yml.j2
│           ├── nginx.conf.j2
│           └── .env.j2
└── site.yml
```

## Шаг 2. Переменные

**roles/app/defaults/main.yml:**

```yaml
---
app_name: myapp
app_version: latest
app_port: 3000
app_domain: app.example.com

# Пути
app_base_dir: /opt/{{ app_name }}
app_releases_dir: "{{ app_base_dir }}/releases"
app_current_link: "{{ app_base_dir }}/current"
app_shared_dir: "{{ app_base_dir }}/shared"

# Docker
app_image: "ghcr.io/myorg/myapp"
app_replicas: 1

# Database
app_db_name: "{{ app_name }}"
app_db_user: "{{ app_name }}"
app_db_password: "{{ vault_db_password }}"

# Healthcheck
app_health_url: "http://localhost:{{ app_port }}/health"
app_health_retries: 10
app_health_delay: 5

# Сколько релизов хранить
app_keep_releases: 5
```

**inventory/group_vars/all/vars.yml:**

```yaml
---
app_version: "v1.2.0"
app_domain: myapp.example.com
```

## Шаг 3. Шаблоны

**templates/compose.yml.j2:**

```yaml
# Managed by Ansible — DO NOT EDIT
services:
  app:
    image: {{ app_image }}:{{ app_version }}
    restart: unless-stopped
    env_file: .env
    ports:
      - "{{ app_port }}:{{ app_port }}"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:{{ app_port }}/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: {{ app_db_name }}
      POSTGRES_USER: {{ app_db_user }}
      POSTGRES_PASSWORD: {{ app_db_password }}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ app_db_user }}"]
      interval: 5s
      timeout: 3s
      retries: 5

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy

volumes:
  db_data:
```

**templates/.env.j2:**

```bash
# Managed by Ansible
NODE_ENV=production
PORT={{ app_port }}
DATABASE_URL=postgres://{{ app_db_user }}:{{ app_db_password }}@db:5432/{{ app_db_name }}
```

**templates/nginx.conf.j2:**

```nginx
server {
    listen 80;
    server_name {{ app_domain }};

    location / {
        proxy_pass http://app:{{ app_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        proxy_pass http://app:{{ app_port }}/health;
        access_log off;
    }
}
```

## Шаг 4. Задачи деплоя

**roles/app/tasks/main.yml:**

```yaml
---
- name: Ensure base directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - "{{ app_base_dir }}"
    - "{{ app_releases_dir }}"
    - "{{ app_shared_dir }}"

- name: Deploy application
  ansible.builtin.include_tasks: deploy.yml
```

**roles/app/tasks/deploy.yml:**

```yaml
---
- name: Set release name
  ansible.builtin.set_fact:
    release_dir: "{{ app_releases_dir }}/{{ ansible_date_time.epoch }}_{{ app_version }}"

- block:
    # === DEPLOY ===
    - name: Create release directory
      ansible.builtin.file:
        path: "{{ release_dir }}"
        state: directory

    - name: Deploy compose.yml
      ansible.builtin.template:
        src: compose.yml.j2
        dest: "{{ release_dir }}/compose.yml"
        mode: '0600'                 # содержит пароли через env_file

    - name: Deploy .env
      ansible.builtin.template:
        src: .env.j2
        dest: "{{ release_dir }}/.env"
        mode: '0600'
      no_log: true                   # не показывать пароли в логе

    - name: Deploy nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: "{{ release_dir }}/nginx.conf"

    - name: Pull images
      ansible.builtin.command:
        cmd: docker compose pull
        chdir: "{{ release_dir }}"
      changed_when: true

    - name: Save previous release path
      ansible.builtin.stat:
        path: "{{ app_current_link }}"
      register: current_link

    - name: Record previous release
      ansible.builtin.set_fact:
        previous_release: "{{ current_link.stat.lnk_target }}"
      when: current_link.stat.exists and current_link.stat.islnk

    - name: Switch symlink to new release
      ansible.builtin.file:
        src: "{{ release_dir }}"
        dest: "{{ app_current_link }}"
        state: link
        force: yes

    - name: Start application
      ansible.builtin.command:
        cmd: docker compose up -d --remove-orphans
        chdir: "{{ app_current_link }}"
      changed_when: true

    - name: Wait for application to be healthy
      ansible.builtin.uri:
        url: "{{ app_health_url }}"
        status_code: 200
      register: health
      until: health.status == 200
      retries: "{{ app_health_retries }}"
      delay: "{{ app_health_delay }}"

    - name: Deploy successful
      ansible.builtin.debug:
        msg: "✅ {{ app_name }}:{{ app_version }} deployed successfully"

  rescue:
    # === ROLLBACK ===
    - name: Deploy FAILED — starting rollback
      ansible.builtin.debug:
        msg: "❌ Deploy failed! Rolling back..."

    - name: Stop failed release
      ansible.builtin.command:
        cmd: docker compose down
        chdir: "{{ release_dir }}"
      ignore_errors: yes

    - name: Rollback symlink
      ansible.builtin.file:
        src: "{{ previous_release }}"
        dest: "{{ app_current_link }}"
        state: link
        force: yes
      when: previous_release is defined

    - name: Restart previous version
      ansible.builtin.command:
        cmd: docker compose up -d
        chdir: "{{ app_current_link }}"
      when: previous_release is defined

    - name: Rollback complete
      ansible.builtin.fail:
        msg: "Rolled back to {{ previous_release | default('unknown') }}. Investigate and retry."

  always:
    # === CLEANUP ===
    - name: Clean old releases (keep {{ app_keep_releases }})
      ansible.builtin.shell: |
        ls -1dt {{ app_releases_dir }}/*/ | tail -n +{{ app_keep_releases + 1 }} | xargs rm -rf
      changed_when: false
      ignore_errors: yes
```

## Шаг 5. Playbook

**site.yml:**

```yaml
---
- name: Deploy Application
  hosts: all
  become: yes
  # serial: 1                       # раскомментировать для rolling update

  roles:
    - app
```

## Шаг 6. Запуск

```bash
# Первый деплой
ansible-playbook site.yml --ask-vault-pass

# Обновление версии
ansible-playbook site.yml --ask-vault-pass -e "app_version=v1.3.0"

# Dry run
ansible-playbook site.yml --check --diff --ask-vault-pass

# Rolling update на нескольких серверах
ansible-playbook site.yml --ask-vault-pass -e "app_version=v1.3.0"
# (раскомментировать serial: 1 в site.yml)
```

Структура на сервере после нескольких деплоев:

```
/opt/myapp/
├── current -> /opt/myapp/releases/1707321600_v1.3.0
├── releases/
│   ├── 1707235200_v1.1.0/
│   ├── 1707278400_v1.2.0/
│   └── 1707321600_v1.3.0/       # текущий
│       ├── compose.yml
│       ├── .env
│       └── nginx.conf
└── shared/
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| block/rescue/always | try/catch/finally — автоматический rollback при ошибке |
| no_log | Пароли не попадают в лог Ansible |
| Atomic symlink | Переключение версии одной операцией |
| Healthcheck | `uri` + `until` + `retries` — ждём готовности |
| Cleanup | Автоматическое удаление старых релизов |
| Extra vars | `-e "app_version=v1.3.0"` — обновление без правки файлов |

## Что дальше

Цепочка tutorials завершена. Рекомендуемые следующие шаги:
- [[ansible/how-to/deploy-strategies]] — canary, blue-green
- [[ansible/how-to/test-with-molecule]] — тестирование ролей
- [[ansible/how-to/manage-secrets]] — vault для CI/CD

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `.env` с паролями в логе | Пароли видны в выводе Ansible | `no_log: true` на задаче с template |
| Нет healthcheck | Rollback не срабатывает (ошибка не обнаружена) | Всегда `uri` + `until` после запуска |
| `docker compose up` без `--remove-orphans` | Старые контейнеры остаются | Добавить `--remove-orphans` |
| Rollback без `previous_release` | Первый деплой — нечего откатывать | `when: previous_release is defined` |