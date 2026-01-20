---
title: "Асинхронные задачи в Ansible"
description: "Запуск долгих операций с async/poll, режим fire-and-forget и ожидание событий с wait_for."
---


По умолчанию Ansible ждет завершения каждой задачи перед переходом к следующей. Это не всегда удобно для долгих операций (обновление системы, скачивание больших файлов).

## Async и Poll
Параметры, управляющие временем выполнения:
* `async`: Максимальное время в секундах, которое задача может работать.
* `poll`: Как часто (в секундах) проверять статус задачи.

### Пример 1: Долгое выполнение с ожиданием
```yaml
- name: Upgrade system packages
  apt: upgrade=dist update_cache=yes
  async: 1200      # Ждать до 20 минут
  poll: 60         # Проверять раз в минуту
```

### Пример 2: Fire-and-Forget (Запустить и забыть)
Если установить `poll: 0`, Ansible запустит задачу и сразу перейдет к следующей, не дожидаясь результата.
```yaml
- name: Run heavy background script
  command: /opt/scripts/heavy_job.sh
  async: 3600
  poll: 0
```

## Ожидание событий (`wait_for`)
Модуль `wait_for` позволяет приостановить выполнение до наступления события (например, порт открылся после ребута).

```yaml
- name: Wait for SSH to come back
  wait_for:
    host: "{{ inventory_hostname }}"
    port: 22
    delay: 10
    timeout: 300
  delegate_to: localhost
```

## Связанные материалы
- [[devops/ansible/how-to/use-handlers|Использование Handlers]]
