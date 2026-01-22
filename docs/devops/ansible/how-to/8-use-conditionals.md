---
title: "8 Условия и логика (When)"
description: "Как использовать when для управления потоком выполнения."
---

Оператор `when` позволяет выполнять задачу только если условие истинно.

## Базовые примеры

```yaml
# Сравнение фактов ОС
- name: Install Apache (Debian/Ubuntu)
  apt: name=apache2 state=present
  when: ansible_os_family == "Debian"

- name: Install HTTPD (RHEL/CentOS)
  yum: name=httpd state=present
  when: ansible_os_family == "RedHat"

# Проверка булевой переменной
- name: Run migration
  command: python manage.py migrate
  when: run_migrations | bool
```

## Использование с Register
Часто нужно выполнить действие в зависимости от результата предыдущей команды.

```yaml
- name: Check if config exists
  stat: path=/etc/app/config.ini
  register: config_file

- name: Init Config
  command: /opt/app/init_config.sh
  when: not config_file.stat.exists
```

## Сложные условия
```yaml
when: 
  - ansible_distribution == "Ubuntu"
  - ansible_memtotal_mb > 1024
  - not config_file.stat.exists
```
(Все условия в списке объединяются через `AND`).
