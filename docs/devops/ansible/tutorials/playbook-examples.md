---
title: "Примеры Ansible Playbooks"
description: "Готовые рецепты: установка LEMP стека, деплой Node.js, настройка безопасности сервера."
---


Коллекция готовых сценариев для частых задач.

## 1. Установка LEMP (Linux, Nginx, MySQL, PHP)

Этот плейбук устанавливает веб-стек и настраивает виртуальный хост.

```yaml
***
- name: Setup LEMP Stack
  hosts: webservers
  become: yes
  vars:
    php_ver: "8.2"
  
  tasks:
    - name: Install Packages
      apt:
        name: 
          - nginx
          - "php{{ php_ver }}-fpm"
          - mysql-server
        state: present
        update_cache: yes

    - name: Start Services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: [nginx, "php{{ php_ver }}-fpm", mysql]

    - name: Configure Nginx
      template:
        src: templates/site.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

## 2. Настройка безопасности (Hardening)

Базовая настройка файрвола (UFW) и SSH.

```yaml
***
- name: Server Hardening
  hosts: all
  become: yes

  tasks:
    - name: Allow SSH, HTTP, HTTPS
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: 

    - name: Enable UFW
      community.general.ufw:
        state: enabled

    - name: Secure SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?PermitRootLogin"
        line: "PermitRootLogin no"
      notify: Restart SSH

  handlers:
    - name: Restart SSH
      service:
        name: ssh
        state: restarted
```

## 3. Обновление системы с перезагрузкой

Обновляет все пакеты и перезагружает сервер, если это необходимо (например, обновилось ядро).

```yaml
***
- name: System Update
  hosts: all
  become: yes

  tasks:
    - name: Upgrade all packages
      apt:
        upgrade: dist
        update_cache: yes

    - name: Check reboot requirement
      stat:
        path: /var/run/reboot-required
      register: reboot_file

    - name: Reboot if needed
      reboot:
        msg: "Rebooting for updates"
      when: reboot_file.stat.exists
```

## Связанные материалы
- [[devops/ansible/explanation/best-practices|Лучшие практики Ansible]]
- [[devops/ansible/reference/common-modules|Справочник модулей]]
