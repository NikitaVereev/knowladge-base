---
title: "Обработка ошибок в Ansible"
description: "Использование блоков block/rescue/always, игнорирование ошибок и кастомные условия сбоя."
---


По умолчанию Ansible останавливает выполнение Playbook на хосте, если задача вернула ошибку. Но это поведение можно изменить.

## Игнорирование ошибок
Если задача не критична (например, проверка существования файла), можно продолжить выполнение.

```yaml
- name: Run command that might fail
  command: /bin/false
  ignore_errors: yes
```

## Блоки (Block / Rescue / Always)
Аналог `try...catch...finally` в программировании. Позволяет группировать задачи и обрабатывать сбои.

```yaml
- block:
    - name: Try to install package
      apt: name=myapp state=present
      
  rescue:
    - name: Rollback (if install failed)
      file: path=/opt/myapp state=absent
      
  always:
    - name: Cleanup temp files
      file: path=/tmp/installer state=absent
```

## Кастомные условия сбоя (`failed_when`)
Иногда команда возвращает код 0, но на самом деле произошла ошибка (или наоборот).

```yaml
- name: Check logs for error
  shell: grep "ERROR" /var/log/app.log
  register: log_check
  # Считать ошибкой только если греп нашел строку "CRITICAL"
  failed_when: "'CRITICAL' in log_check.stdout"
```

## Связанные материалы
- [[devops/ansible/how-to/debug-playbook|Отладка Playbooks]]
