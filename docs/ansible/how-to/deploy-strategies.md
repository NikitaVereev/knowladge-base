---
title: "Стратегии деплоя"
type: how-to
tags: [ansible, deploy, rolling, blue-green, canary, serial, zero-downtime, rollback]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html"
related:
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/explanation/error-handling]]"
  - "[[ansible/how-to/write-playbooks]]"
---

# Стратегии деплоя с Ansible

> **TL;DR:** `serial: 1` — rolling update (по одному). `serial: [1, "100%"]` — canary.
> Blue-green — через atomic symlink. Всегда: healthcheck + вывод из LB + откат.

## 1. Rolling Deployment (Поэтапное обновление)

Обновляем серверы по очереди. Самый простой и надёжный метод.

```yaml
- name: Rolling Deployment
  hosts: webservers
  serial: 1                          # по 1 хосту за раз
  max_fail_percentage: 20            # остановить если >20% упало
  become: yes

  pre_tasks:
    - name: Вывести из балансировщика
      ansible.builtin.uri:
        url: "http://lb.local/api/disable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

  tasks:
    - name: Stop service
      ansible.builtin.systemd:
        name: myapp
        state: stopped

    - name: Update code
      ansible.builtin.git:
        repo: "https://github.com/org/app.git"
        dest: /opt/app
        version: "{{ app_version }}"

    - name: Install dependencies
      ansible.builtin.command: npm ci --production
      args:
        chdir: /opt/app

    - name: Start service
      ansible.builtin.systemd:
        name: myapp
        state: started

    - name: Health check
      ansible.builtin.uri:
        url: http://localhost:8080/health
        status_code: 200
      register: health
      until: health.status == 200
      retries: 10
      delay: 5

  post_tasks:
    - name: Вернуть в балансировщик
      ansible.builtin.uri:
        url: "http://lb.local/api/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost
```

### Варианты `serial`

```yaml
serial: 1              # по 1 хосту
serial: 3              # по 3 хоста
serial: "30%"          # по 30% от группы
serial:                # прогрессивный
  - 1                  # сначала 1 (canary)
  - 5                  # потом 5
  - "100%"             # потом все остальные
```

## 2. Blue-Green Deployment

Два идентичных окружения. Деплоим в «зелёное», переключаем трафик атомарно.

### Вариант: atomic symlink

```yaml
vars:
  app_dir: /opt/app
  releases_dir: "{{ app_dir }}/releases"
  release_name: "{{ ansible_date_time.epoch }}"
  release_path: "{{ releases_dir }}/{{ release_name }}"
  current_path: "{{ app_dir }}/current"

tasks:
  - name: Создать директорию релиза
    ansible.builtin.file:
      path: "{{ release_path }}"
      state: directory

  - name: Деплой новой версии (Green)
    ansible.builtin.git:
      repo: "https://github.com/org/app.git"
      dest: "{{ release_path }}"
      version: "{{ app_version }}"

  - name: Установить зависимости
    ansible.builtin.command: npm ci --production
    args:
      chdir: "{{ release_path }}"

  - name: Smoke test (проверить до переключения)
    ansible.builtin.command: node {{ release_path }}/healthcheck.js
    delegate_to: localhost

  - name: Переключить трафик (atomic symlink)
    ansible.builtin.file:
      src: "{{ release_path }}"
      dest: "{{ current_path }}"
      state: link
    notify: Reload app

  - name: Удалить старые релизы (оставить 5)
    ansible.builtin.shell: |
      ls -1dt {{ releases_dir }}/*/ | tail -n +6 | xargs rm -rf
    args:
      warn: false
    changed_when: false
```

### Откат

```bash
# Быстрый откат — переключить symlink на предыдущий релиз
ansible webservers -m file -a \
  "src=/opt/app/releases/PREVIOUS dest=/opt/app/current state=link" -b
```

## 3. Canary Deployment

Выкатка на малую часть серверов → проверка метрик → полная выкатка.

```yaml
- name: Canary Release
  hosts: webservers
  serial:
    - 1                              # Phase 1: один сервер
    - "100%"                         # Phase 2: все остальные

  tasks:
    - name: Deploy application
      ansible.builtin.apt:
        name: myapp
        state: latest

    - name: Health check
      ansible.builtin.uri:
        url: http://localhost:8080/health
        status_code: 200
      retries: 5
      delay: 3

    - name: Пауза после canary (только после 1-го батча)
      ansible.builtin.pause:
        prompt: |
          Canary deployed to {{ inventory_hostname }}.
          Check metrics at http://grafana.local/dashboard
          Press Enter to continue or Ctrl+C to abort...
      when: ansible_play_batch | length == 1
      run_once: true
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `serial` не задан | Все серверы обновляются одновременно → полный downtime | `serial: 1` минимум |
| Нет healthcheck | Сломанная версия на всех серверах | `uri` + `until` + `retries` после каждого деплоя |
| Нет вывода из LB | Трафик идёт на сервер во время обновления → 502 | `pre_tasks` для вывода из балансировщика |
| Откат не предусмотрен | Паника при проблемах | Symlink-подход: `ln -sfn /releases/old /current` |
| `pause` без `run_once` | Пауза для каждого хоста в батче | `run_once: true` для canary pause |