---
title: "Отладка Ansible Playbook"
description: "Использование модуля debug, интерактивного дебаггера и флага --diff."
---


Когда что-то идет не так, Ansible предоставляет инструменты для инспекции.

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
Можно поставить "точку останова" при ошибке, чтобы изучить состояние системы.

```yaml
- name: Task that fails
  service: name=httpd state=restarted
  debugger: on_failed
```

**Команды в консоли:**
* `p task_vars` — показать переменные.
* `p result` — показать результат выполнения.
* `redo` — повторить задачу.
* `continue` — продолжить.

## Режим Diff
Флаг `--diff` показывает, **что именно** изменилось в файлах (как `git diff`).

```bash
ansible-playbook site.yml --diff
```

## Связанные материалы
- [[devops/ansible/explanation/error-handling|Обработка ошибок]]
- [[devops/ansible/how-to/run-playbook|Запуск Playbook]]
