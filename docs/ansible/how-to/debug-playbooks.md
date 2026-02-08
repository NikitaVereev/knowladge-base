---
title: "Отладка Playbooks"
type: how-to
tags: [ansible, debug, verbosity, diff, debugger, assert, callback, troubleshooting]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_debugger.html"
related:
  - "[[ansible/explanation/error-handling]]"
  - "[[ansible/how-to/write-playbooks]]"
  - "[[ansible/reference/cheatsheet]]"
---

# Отладка Ansible Playbooks

> **TL;DR:** `debug` модуль → вывод переменных. `-vvv` → детали SSH. `--diff` → что изменилось.
> `debugger: on_failed` → интерактивная точка останова. `assert` → проверка предусловий.

## Модуль debug

```yaml
# Вывести значение переменной
- name: Show variable
  ansible.builtin.debug:
    msg: "App port is {{ app_port }}, version {{ app_version }}"

# Вывести структуру переменной (dict, list)
- name: Show facts
  ansible.builtin.debug:
    var: ansible_default_ipv4

# Вывести результат предыдущей задачи
- name: Run command
  ansible.builtin.command: python3 --version
  register: py_version
  changed_when: false

- name: Show result
  ansible.builtin.debug:
    var: py_version.stdout
```

## Флаги командной строки

```bash
# Diff — показать что изменилось в файлах (как git diff)
ansible-playbook site.yml --diff

# Check + Diff — идеальная предпроверка
ansible-playbook site.yml --check --diff

# Verbosity
ansible-playbook site.yml -v       # результаты задач
ansible-playbook site.yml -vv      # входные параметры модулей
ansible-playbook site.yml -vvv     # SSH-подключения, пути к файлам
ansible-playbook site.yml -vvvv    # скрипты, передаваемые на хост

# Пошаговый режим (подтверждение каждой задачи)
ansible-playbook site.yml --step

# Начать с конкретной задачи
ansible-playbook site.yml --start-at-task "Deploy application"

# Показать список задач/тегов/хостов без выполнения
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-tags
ansible-playbook site.yml --list-hosts

# Syntax check (без подключения к хостам)
ansible-playbook site.yml --syntax-check
```

## Интерактивный Debugger

Точка останова при ошибке — изучить состояние прямо во время выполнения.

```yaml
- name: Task that might fail
  ansible.builtin.service:
    name: myapp
    state: restarted
  debugger: on_failed              # открыть debugger при ошибке
```

Уровни: `always`, `never`, `on_failed`, `on_unreachable`, `on_skipped`.

Команды в debugger:

| Команда | Действие |
|---------|----------|
| `p task_vars` | Показать все переменные |
| `p task_vars['app_port']` | Показать конкретную переменную |
| `p result` | Показать результат (ошибку) |
| `p task.args` | Показать аргументы задачи |
| `task.args['name'] = 'nginx'` | Изменить аргумент |
| `redo` | Повторить задачу (с изменёнными аргументами) |
| `continue` | Продолжить выполнение |
| `quit` | Завершить playbook |

## Assert (проверка предусловий)

```yaml
- name: Verify prerequisites
  ansible.builtin.assert:
    that:
      - ansible_memtotal_mb >= 1024
      - ansible_distribution == "Ubuntu"
      - app_version is defined
    fail_msg: "Server doesn't meet requirements"
    success_msg: "All checks passed"
```

## Ansible Lint

```bash
# Установка
pipx inject ansible ansible-lint

# Проверка
ansible-lint site.yml
ansible-lint roles/

# Игнорировать правило для задачи
- name: Run shell command
  ansible.builtin.shell: echo "hello"     # noqa: command-instead-of-shell
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Не используют `--diff` | «Что-то поменялось, но что?» | Всегда `--diff` при отладке template/lineinfile |
| `debug` задачи в production | Лишний вывод, замедление | Оборачивать в `when: debug_mode \| default(false)` или `tags: [debug, never]` |
| `-vvvv` на 100 хостах | Гигабайты логов | Комбинировать с `--limit web-01` |
| Не знают про `--syntax-check` | Ошибки YAML обнаруживаются только при запуске | `ansible-playbook --syntax-check` + `ansible-lint` в CI |