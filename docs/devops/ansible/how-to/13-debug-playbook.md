---
title: "13 Отладка Ansible Playbook"
description: "Использование модуля debug, интерактивного дебаггера и флага --diff."
---

## Модуль `debug`
Самый простой способ вывести значения переменных.

```yaml
- name: Print variable
  debug:
    msg: "The app port is {{ app_port }}"

- name: Print full variable structure
  debug:
    var: ansible_facts
```

## Интерактивный Debugger
Можно поставить "точку останова" при ошибке, чтобы изучить состояние системы прямо во время выполнения.

```yaml
- name: Task that fails
  service: name=httpd state=restarted
  debugger: on_failed
```

Когда задача упадет, Ansible откроет консоль `[root@host] >`.
*   `p task_vars` — показать переменные.
*   `p result` — показать ошибку.
*   `redo` — повторить задачу.

## Режим Diff
Флаг `--diff` показывает, **что именно** изменилось в файлах (как `git diff`). Очень полезно для модулей `template` и `lineinfile`.

```bash
ansible-playbook site.yml --diff
```
