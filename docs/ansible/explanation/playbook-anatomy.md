---
title: "Анатомия Playbook"
type: explanation
tags: [ansible, playbook, play, tasks, handlers, pre-tasks, post-tasks, order-of-execution]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/index.html"
related:
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/explanation/roles-and-collections]]"
  - "[[ansible/how-to/write-playbooks]]"
  - "[[ansible/tutorials/01-first-playbook]]"
---

# Анатомия Ansible Playbook

> **TL;DR:** Playbook = список Plays. Play связывает группу хостов с задачами.
> Порядок: Gather Facts → pre_tasks → roles → tasks → post_tasks → handlers.

Playbook — сценарий автоматизации в формате YAML. Он описывает **что** делать (Tasks) и **где** это делать (Hosts).

## Структура Playbook

Один файл может содержать несколько сценариев (Plays). Каждый Play применяет набор задач к группе хостов.

```yaml
---
- name: Настройка веб-серверов        # ← Play 1
  hosts: webservers                    # ← к каким хостам применять
  become: yes                          # ← выполнять от root (sudo)
  vars:
    http_port: 80

  tasks:
    - name: Установить Nginx           # ← Task 1
      ansible.builtin.apt:
        name: nginx
        state: present
      notify: Restart Nginx            # ← триггер handler'а

    - name: Скопировать конфиг         # ← Task 2
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx              # ← выполнится один раз в конце
      ansible.builtin.service:
        name: nginx
        state: restarted

- name: Настройка базы данных          # ← Play 2
  hosts: db
  become: yes
  tasks:
    - name: Установить PostgreSQL
      ansible.builtin.apt:
        name: postgresql
        state: present
```

## Основные компоненты

### 1. Play (Сценарий)
Связывает группу хостов с задачами. Определяет контекст выполнения.

Ключевые директивы Play:

| Директива | Назначение |
|-----------|-----------|
| `hosts:` | Группа хостов из inventory |
| `become: yes` | Повышение привилегий (sudo) |
| `vars:` | Переменные для этого Play |
| `gather_facts: no` | Отключить сбор фактов (ускоряет выполнение) |
| `serial: 2` | Выполнять на N хостах за раз (rolling update) |
| `max_fail_percentage: 25` | Допустимый процент ошибок |
| `any_errors_fatal: true` | Остановить всё при первой ошибке |

### 2. Tasks (Задачи)
Список действий, выполняемых **последовательно сверху вниз**. Каждая задача вызывает один модуль Ansible. Если задача падает — выполнение для этого хоста останавливается.

```yaml
tasks:
  - name: Описание задачи          # всегда давайте понятное имя
    ansible.builtin.apt:            # FQCN модуля (рекомендуется)
      name: nginx
      state: present
    become: yes                     # можно на уровне задачи
    when: ansible_os_family == "Debian"   # условие выполнения
    tags: [install, nginx]          # теги для выборочного запуска
    register: install_result        # сохранить результат
```

### 3. Handlers (Обработчики)
Специальные задачи, которые запускаются:
- **Только если** их уведомили (`notify`)
- **Только если** уведомляющая задача внесла изменения (`changed`)
- **Один раз** в конце Play (даже если notify вызван несколько раз)

Handler «Restart Nginx» выполнится единожды, даже если и `apt` и `template` вернули `changed`.

### 4. Pre/Post Tasks
Задачи, выполняемые до и после ролей.

```yaml
- hosts: webservers
  pre_tasks:
    - name: Вывести из балансировщика
      uri: url=http://lb/remove/{{ inventory_hostname }}

  roles:
    - nginx
    - app

  post_tasks:
    - name: Вернуть в балансировщик
      uri: url=http://lb/add/{{ inventory_hostname }}
```

## Порядок выполнения

```
1. Gather Facts       — сбор фактов (ansible_os_family, ansible_default_ipv4...)
2. pre_tasks          — подготовительные задачи
   └── handlers       — (flush если notify в pre_tasks)
3. roles              — задачи из подключённых ролей
   └── handlers       — (flush)
4. tasks              — основные задачи playbook
   └── handlers       — (flush)
5. post_tasks         — завершающие задачи
   └── handlers       — (flush)
```

Handler'ы можно вызвать принудительно в середине выполнения:

```yaml
tasks:
  - name: Install Nginx
    apt: name=nginx state=present
    notify: Restart Nginx

  - meta: flush_handlers            # ← Restart Nginx выполнится ЗДЕСЬ, а не в конце

  - name: Verify Nginx is running
    uri: url=http://localhost
```

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «Handler выполняется сразу после notify» | Handler выполняется в конце Play. Используй `meta: flush_handlers` если нужно раньше |
| «Задачи выполняются параллельно» | Задачи — последовательно. Параллельность — между хостами (`forks`) |
| «`name:` необязателен» | Технически да, но без name логи нечитаемы. Всегда описывай задачу |
| «`become: yes` на уровне Play — для всех задач» | Да, но можно переопределить `become: no` на уровне конкретной задачи |