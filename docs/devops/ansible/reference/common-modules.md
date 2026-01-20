---
title: "Популярные модули Ansible"
description: "Шпаргалка по модулям: apt, yum, copy, template, file, service, user, git."
---


Краткий справочник по модулям, которые используются в 90% задач.

## Управление пакетами
**apt (Debian/Ubuntu):**
```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
```

**dnf/yum (RHEL/CentOS):**
```yaml
- name: Install git
  dnf:
    name: git
    state: latest
```

## Файлы и Директории
**file (создание папок, права):**
```yaml
- name: Create directory
  file:
    path: /opt/myapp
    state: directory
    mode: '0755'
    owner: www-data
```

**copy (копирование файлов):**
```yaml
- name: Copy config
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    backup: yes
```

**template (шаблонизация Jinja2):**
```yaml
- name: Configure app
  template:
    src: templates/config.j2
    dest: /etc/app/config.ini
```

## Управление сервисами
**systemd:**
```yaml
- name: Start and enable nginx
  systemd:
    name: nginx
    state: started
    enabled: yes
```

## Выполнение команд
**shell (с поддержкой pipe/redirect):**
```yaml
- name: Check process
  shell: "ps aux | grep nginx"
  register: ps_result
```

**command (безопаснее, без shell-окружения):**
```yaml
- name: Run script
  command: /opt/scripts/deploy.sh
```

## Связанные материалы
- [[devops/ansible/how-to/ad-hoc-commands|Ad-Hoc команды]]
