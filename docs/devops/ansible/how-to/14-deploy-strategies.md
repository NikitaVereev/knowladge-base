---
title: "14 Стратегии деплоя с Ansible"
description: "Реализация стратегий Rolling Update, Blue-Green и Canary деплоя."
---

Ansible позволяет реализовать сложные стратегии развертывания для обеспечения Zero Downtime.

## 1. Rolling Deployment (Поэтапное обновление)
Самый простой метод: обновляем серверы по очереди.

```yaml
- name: Rolling Deployment
  hosts: webservers
  serial: 1  # Или "30%" — количество хостов за раз
  max_fail_percentage: 20 # Остановить, если >20% упало
  
  tasks:
    - name: Disable in Load Balancer
      # Пример вызова API балансировщика
      uri:
        url: "http://lb.local/api/disable?host={{ inventory_hostname }}"
        method: POST

    - name: Stop Service
      service: name=myapp state=stopped

    - name: Update Code
      git: repo=... dest=/opt/app version=main

    - name: Start Service
      service: name=myapp state=started

    - name: Health Check
      uri:
        url: http://localhost:8080/health
        status_code: 200
      register: result
      until: result.status == 200
      retries: 10
      delay: 5

    - name: Enable in Load Balancer
      uri:
        url: "http://lb.local/api/enable?host={{ inventory_hostname }}"
        method: POST
```

## 2. Blue-Green Deployment
Метод переключения трафика. Требует наличия двух идентичных окружений (или директорий) и переключателя (обычно симлинк или Nginx).

**Вариант с симлинками (на одном сервере):**

```yaml
vars:
  release_path: "/opt/app/releases/{{ ansible_date_time.iso8601 }}"
  current_path: "/opt/app/current"

tasks:
  - name: 1. Deploy New Version (Green)
    git:
      repo: ...
      dest: "{{ release_path }}"
      version: main

  - name: 2. Install Dependencies
    command: npm install
    args:
      chdir: "{{ release_path }}"

  - name: 3. Smoke Test (Проверка новой версии)
    command: npm test
    args:
      chdir: "{{ release_path }}"

  - name: 4. Switch Traffic (Atomic Symlink)
    file:
      src: "{{ release_path }}"
      dest: "{{ current_path }}"
      state: link
    notify: Reload Nginx
```

## 3. Canary Deployment (Канарейка)
Выкатка на малую часть пользователей (или серверов) для проверки боем.

```yaml
- name: Canary Release
  hosts: webservers
  serial:
    - 1        # Сначала только 1 сервер
    - "100%"   # Если всё ок, то остальные
  
  tasks:
    - name: Update App
      yum: name=myapp state=latest
    
    - name: Manual Confirmation (Pause)
      pause:
        prompt: "Canary node updated. Check metrics. Press Enter to continue deployment..."
      # Пауза сработает только после первого батча (1 сервер)
      when: inventory_hostname == play_hosts
```
