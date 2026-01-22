---
title: "5 Переменные и Facts в Ansible"
description: "Типы переменных, приоритеты (Variable Precedence) и автоматический сбор фактов."
---

Ansible использует переменные для управления различиями между системами. Они могут быть определены в десятках мест, поэтому важно понимать порядок их применения.

## Типы переменных

### 1. Пользовательские переменные
Задаются вами в Playbook, Inventory или отдельных файлах (`group_vars`, `host_vars`).
```yaml
vars:
  app_version: "1.2.0"
  db_port: 5432
```

### 2. Facts (Факты)
Информация, которую Ansible собирает о хосте перед выполнением задач (модуль `setup`).
*   `ansible_distribution`: Ubuntu/CentOS
*   `ansible_memtotal_mb`: Объем RAM
*   `ansible_default_ipv4.address`: Основной IP адрес
*   `ansible_processor_vcpus`: Количество ядер

### 3. Register (Регистры)
Переменные, создаваемые динамически из результата выполнения задачи.
```yaml
- shell: cat /etc/passwd
  register: passwd_contents
```

## Приоритет переменных (Variable Precedence)
От наименьшего к наибольшему (последний выигрывает и перезаписывает предыдущие):

1.  Inventory group_vars (all)
2.  Inventory group_vars (specific group)
3.  Inventory host_vars
4.  Playbook group_vars
5.  Playbook host_vars
6.  Play vars (внутри `vars:` секции плейбука)
7.  Task vars (внутри `vars:` секции задачи)
8.  **Extra vars** (флаг `-e` в командной строке) — **Всегда побеждает!**
    `ansible-playbook site.yml -e "force_upgrade=true"`
