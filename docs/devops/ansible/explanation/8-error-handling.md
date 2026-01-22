---
title: "8 Обработка ошибок в Ansible"
description: "Использование блоков block/rescue/always, игнорирование ошибок и кастомные условия сбоя."
---

По умолчанию Ansible останавливает выполнение Playbook на хосте, если задача вернула ошибку (return code != 0).

## Игнорирование ошибок
Если задача не критична (например, попытка удалить файл, которого может не быть), можно продолжить выполнение.

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
    - name: Configure package
      template: src=conf.j2 dest=/etc/myapp.conf

  rescue:
    - name: Rollback (выполнится только при ошибке в block)
      file: path=/opt/myapp state=absent
    - name: Notify admin
      debug: msg="Installation failed, rolled back."

  always:
    - name: Cleanup temp files (выполнится всегда)
      file: path=/tmp/installer state=absent
```

## Кастомные условия сбоя (`failed_when`)
Иногда команда возвращает код 0 (успех), но на самом деле произошла логическая ошибка (например, в выводе stderr есть слово "Error").

```yaml
- name: Check logs for error
  shell: grep "ERROR" /var/log/app.log
  register: log_check
  # Считать ошибкой только если греп нашел строку "CRITICAL"
  # (по умолчанию grep возвращает 0 если нашел, и 1 если нет)
  failed_when: "'CRITICAL' in log_check.stdout"
```
