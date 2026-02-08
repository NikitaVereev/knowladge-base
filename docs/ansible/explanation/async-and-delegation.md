---
title: "Асинхронные задачи и делегирование"
type: explanation
tags: [ansible, async, poll, fire-and-forget, delegate_to, wait_for, run_once, local_action]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_async.html"
related:
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/explanation/error-handling]]"
  - "[[ansible/how-to/write-playbooks]]"
---

# Асинхронные задачи и делегирование

> **TL;DR:** `async` + `poll` — для долгих операций (обновление ОС, тяжёлые скрипты).
> `poll: 0` — fire-and-forget. `delegate_to` — выполнить задачу на другом хосте.
> `run_once` — выполнить один раз на всю группу.

По умолчанию Ansible ждёт завершения каждой задачи перед переходом к следующей. Это не всегда удобно для долгих операций или когда SSH-сессия может оборваться.

## Async и Poll

Параметры, управляющие временем выполнения:

- `async` — максимальное время в секундах (таймаут).
- `poll` — интервал проверки статуса (в секундах).

### Долгое выполнение с ожиданием

Ansible запустит задачу, закроет SSH и будет переподключаться каждые 60 секунд.

```yaml
- name: Обновление системных пакетов
  ansible.builtin.apt:
    upgrade: dist
    update_cache: yes
  async: 3600        # ждать до 1 часа
  poll: 60           # проверять раз в минуту
```

### Fire-and-Forget (Запустить и забыть)

`poll: 0` — Ansible запустит задачу и **сразу** перейдёт к следующей.

```yaml
- name: Запустить тяжёлый фоновый скрипт
  ansible.builtin.command: /opt/scripts/heavy_job.sh
  async: 3600
  poll: 0
  register: heavy_job

# Позже можно проверить статус
- name: Проверить завершение скрипта
  async_status:
    jid: "{{ heavy_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 30
  delay: 60
```

## Делегирование (delegate_to)

Выполнить задачу не на целевом хосте, а на другом (обычно на localhost или балансировщике).

```yaml
# Вывести сервер из балансировщика перед обновлением
- name: Remove from load balancer
  ansible.builtin.uri:
    url: "http://lb.example.com/api/remove/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost

- name: Update application
  ansible.builtin.apt:
    name: myapp
    state: latest

# Вернуть в балансировщик
- name: Add back to load balancer
  ansible.builtin.uri:
    url: "http://lb.example.com/api/add/{{ inventory_hostname }}"
    method: POST
  delegate_to: localhost
```

### local_action (сокращение для delegate_to: localhost)

```yaml
- name: Сохранить бэкап локально
  local_action:
    module: ansible.builtin.copy
    content: "{{ db_dump.stdout }}"
    dest: "/backups/{{ inventory_hostname }}.sql"
```

## run_once

Выполнить задачу один раз для всей группы (например, миграция БД).

```yaml
- name: Выполнить миграции БД
  ansible.builtin.command: /app/manage.py migrate
  run_once: true        # выполнится только на первом хосте группы
```

## Ожидание событий (wait_for)

Приостановить выполнение до наступления условия.

```yaml
# Ждать пока SSH вернётся после reboot
- name: Перезагрузить сервер
  ansible.builtin.reboot:
    reboot_timeout: 300

# Или вручную:
- name: Reboot
  ansible.builtin.command: shutdown -r now
  async: 1
  poll: 0

- name: Ждать SSH
  ansible.builtin.wait_for:
    host: "{{ ansible_host }}"
    port: 22
    delay: 10
    timeout: 300
    search_regex: OpenSSH
  delegate_to: localhost

# Ждать пока порт приложения откроется
- name: Ждать запуска приложения
  ansible.builtin.wait_for:
    port: 8080
    delay: 5
    timeout: 60
```

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «async ускоряет выполнение» | async не параллелит задачи. Он позволяет не держать SSH-соединение для долгих операций |
| «fire-and-forget надёжен» | Без `async_status` вы не узнаете об ошибке. Всегда проверяй результат |
| «delegate_to сохраняет контекст хоста» | Facts и переменные остаются от целевого хоста, но задача выполняется на делегированном |
| «run_once = выполнить на localhost» | run_once выполняет на первом хосте группы, не на localhost. Комбинируй с `delegate_to: localhost` если нужно |
