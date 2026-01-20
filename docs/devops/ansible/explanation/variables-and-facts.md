---
title: "Переменные и Facts в Ansible"
description: "Как работают переменные, приоритеты (precedence) и автоматический сбор фактов о системе."
---


Ansible использует переменные для управления различиями между системами. Они могут быть определены в десятках мест, поэтому важно понимать порядок их применения.

## Типы переменных

### 1. Пользовательские переменные
Задаются вами в Playbook, Inventory или отдельных файлах (`group_vars`, `host_vars`).
```yaml
vars:
  http_port: 80
  app_version: "1.2.0"
```

### 2. Facts (Факты)
Информация, которую Ansible собирает о хосте перед выполнением задач.
* `ansible_distribution`: Ubuntu/CentOS
* `ansible_memtotal_mb`: Объем RAM
* `ansible_default_ipv4.address`: IP адрес

> **Совет:** Используйте команду `ansible localhost -m setup`, чтобы увидеть все доступные факты.

### 3. Register (Регистры)
Переменные, создаваемые динамически из результата выполнения задачи.
```yaml
- shell: cat /etc/passwd
  register: passwd_contents
```

## Приоритет переменных (Variable Precedence)
От наименьшего к наибольшему (последний выигрывает):

1. **Inventory vars** (в файле hosts.ini)
2. **Playbook group_vars**
3. **Playbook host_vars**
4. **Play vars** (внутри playbook)
5. **Task vars** (внутри задачи)
6. **Extra vars** (флаг `-e` в командной строке) — **Всегда побеждает!**

## Связанные материалы
- [[devops/ansible/how-to/use-conditionals|Использование условий и циклов]]
- [[devops/ansible/explanation/inventory-basics|Инвентарь и группы]]
