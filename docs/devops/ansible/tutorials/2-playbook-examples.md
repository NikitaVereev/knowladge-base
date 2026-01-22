---
title: "2 Сборник рецептов Ansible"
description: "Готовые паттерны: LEMP стек, Hardening сервера и автоматическое обновление."
---

Коллекция проверенных решений для частых задач.

## 1. Веб-стек (LEMP)

Установка Nginx, MySQL и PHP с проверкой сервисов.

```yaml
- name: Setup LEMP Stack
  hosts: webservers
  become: yes
  vars:
    php_ver: "8.1"
  
  tasks:
    - name: Установка пакетов
      apt:
        name: 
          - nginx
          - "php{{ php_ver }}-fpm"
          - mysql-server
          - python3-pymysql # Для модуля mysql_db
        state: present
        update_cache: yes

    - name: Запуск сервисов
      service:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: 
        - nginx
        - "php{{ php_ver }}-fpm"
        - mysql

    - name: Удаление дефолтного конфига Nginx
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      service: name=nginx state=reloaded
```

## 2. Hardening (Базовая безопасность)

Настройка Firewall (UFW) и SSH.

```yaml
- name: Server Hardening
  hosts: all
  become: yes
  
  tasks:
    - name: Установка UFW
      apt: name=ufw state=present

    - name: Разрешить SSH, HTTP, HTTPS
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop:[1]

    - name: Включить UFW
      community.general.ufw:
        state: enabled
        policy: deny

    - name: Отключить вход root по SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?PermitRootLogin"
        line: "PermitRootLogin no"
        validate: 'sshd -t -f %s' # Проверка синтаксиса перед сохранением
      notify: Restart SSH

  handlers:
    - name: Restart SSH
      service: name=ssh state=restarted
```

## 3. Умное обновление системы

Обновление только если нужно, с перезагрузкой только при обновлении ядра.

```yaml
- name: Smart Update
  hosts: all
  become: yes
  
  tasks:
    - name: Обновление кэша apt
      apt: update_cache=yes

    - name: Full Upgrade
      apt: upgrade=dist

    - name: Проверка на необходимость ребута
      stat:
        path: /var/run/reboot-required
      register: reboot_file

    - name: Перезагрузка (если требуется)
      reboot:
        msg: "Rebooting due to kernel updates"
        pre_reboot_delay: 10
      when: reboot_file.stat.exists
```
