---
title: "Обработка ошибок"
type: explanation
tags: [ansible, errors, block, rescue, always, ignore-errors, failed-when, changed-when, any-errors-fatal, retry]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html"
related:
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/explanation/async-and-delegation]]"
  - "[[ansible/how-to/debug-playbooks]]"
---

# Обработка ошибок в Ansible

> **TL;DR:** По умолчанию Ansible останавливается при первой ошибке.
> `ignore_errors` — продолжить. `block/rescue/always` — try/catch/finally.
> `failed_when` / `changed_when` — кастомные условия. `any_errors_fatal` — остановить ВСЁ.

По умолчанию Ansible останавливает выполнение Playbook на хосте, если задача вернула ошибку (return code != 0).

## Игнорирование ошибок

```yaml
# Продолжить выполнение даже если задача упала
- name: Удалить файл (может не существовать)
  ansible.builtin.file:
    path: /tmp/maybe-exists
    state: absent
  ignore_errors: yes

# Лучше: ignore_unreachable для проблем с подключением
- name: Проверить доступность
  ansible.builtin.ping:
  ignore_unreachable: yes
```

## Блоки (block / rescue / always)

Аналог `try...catch...finally` в программировании.

```yaml
- block:
    - name: Установить пакет
      ansible.builtin.apt:
        name: myapp
        state: present

    - name: Настроить пакет
      ansible.builtin.template:
        src: conf.j2
        dest: /etc/myapp.conf

  rescue:
    # Выполнится ТОЛЬКО при ошибке в block
    - name: Откатить установку
      ansible.builtin.apt:
        name: myapp
        state: absent

    - name: Уведомить админа
      ansible.builtin.debug:
        msg: "Установка провалилась, выполнен откат"

  always:
    # Выполнится ВСЕГДА (и при успехе, и при ошибке)
    - name: Очистить временные файлы
      ansible.builtin.file:
        path: /tmp/installer
        state: absent
```

### Практический пример: деплой с откатом

```yaml
- block:
    - name: Деплой новой версии
      ansible.builtin.unarchive:
        src: "app-{{ new_version }}.tar.gz"
        dest: /opt/app/
        remote_src: yes

    - name: Применить миграции
      ansible.builtin.command: /opt/app/migrate.sh

    - name: Перезапустить сервис
      ansible.builtin.systemd:
        name: myapp
        state: restarted

    - name: Проверить здоровье
      ansible.builtin.uri:
        url: http://localhost:8080/health
        status_code: 200
      retries: 5
      delay: 3

  rescue:
    - name: Вернуть предыдущую версию
      ansible.builtin.command: >
        ln -sfn /opt/app/releases/{{ old_version }} /opt/app/current

    - name: Перезапустить со старой версией
      ansible.builtin.systemd:
        name: myapp
        state: restarted
```

## Кастомные условия

### failed_when

Переопределить, когда задача считается проваленной.

```yaml
# Команда возвращает 0, но в выводе есть ошибка
- name: Проверить логи
  ansible.builtin.shell: grep "ERROR" /var/log/app.log
  register: log_check
  failed_when: "'CRITICAL' in log_check.stdout"

# Считать успехом и exit code 0, и exit code 2
- name: Diff конфигов
  ansible.builtin.command: diff /etc/app.conf /tmp/new.conf
  register: diff_result
  failed_when: diff_result.rc > 1    # 0=same, 1=different, 2=error
```

### changed_when

Переопределить, когда задача считается изменившей состояние.

```yaml
# shell/command всегда возвращают changed — исправляем
- name: Проверить версию
  ansible.builtin.command: myapp --version
  register: version_check
  changed_when: false                 # эта команда ничего не меняет

# Изменение только при определённом выводе
- name: Применить SQL-миграции
  ansible.builtin.command: /app/migrate.sh
  register: migrate_result
  changed_when: "'Applied' in migrate_result.stdout"
```

## Retry (Повторные попытки)

```yaml
# Повторять задачу до успеха
- name: Ждать готовности API
  ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
  register: health
  until: health.status == 200
  retries: 10              # количество попыток
  delay: 5                 # секунд между попытками
```

## Контроль на уровне Play

```yaml
- hosts: webservers
  serial: 2                        # обновлять по 2 хоста за раз
  max_fail_percentage: 25          # остановиться если 25%+ хостов упали
  any_errors_fatal: true           # ИЛИ: одна ошибка = остановить ВСЁ

  tasks:
    - name: Deploy
      # ...
```

| Директива | Поведение |
|-----------|----------|
| `any_errors_fatal: true` | Одна ошибка на одном хосте → останов всего Play |
| `max_fail_percentage: 25` | Допустимый процент ошибок (при `serial`) |
| `ignore_errors: yes` | Продолжить этот хост, даже если задача упала |
| `ignore_unreachable: yes` | Продолжить, даже если хост недоступен |

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «ignore_errors — нормальная практика» | Маскирует реальные проблемы. Используй `failed_when` для точного контроля |
| «rescue = automatic rollback» | Rescue — просто блок задач. Писать логику отката нужно самому |
| «shell/command показывают changed = правда» | Они всегда `changed`. Используй `changed_when: false` для read-only команд |
| «retries гарантируют успех» | Если задача падает после всех retries — Play падает. Комбинируй с `ignore_errors` или `rescue` |