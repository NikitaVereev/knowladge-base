---
title: 04 Асинхронные Задачи и Handlers
---

---

## Async/Poll

```yaml
- name: Долгая операция
  shell: "long_running_task"
  async: 300              # max время (сек)
  poll: 10                # проверять каждые 10 сек
```

**Параметры:**
- `async` — максимальное время выполнения (секунды)
- `poll` — интервал проверки (0 = не проверять)

---

## Примеры Async

**С ожиданием:**
```yaml
- shell: "apt-get update && apt-get upgrade -y"
  async: 600              # 10 минут max
  poll: 10                # проверять каждые 10 сек
```

**Без ожидания (фоновая):**
```yaml
- shell: "sleep 1000"
  async: 1200
  poll: 0                 # не ждать результат
```

**Перезагрузка сервера:**
```yaml
- reboot:
    reboot_timeout: 600

- wait_for:
    host: "{{ inventory_hostname }}"
    port: 22
    delay: 30
    timeout: 300
```

---

## Wait_for

```yaml
# Дождаться загрузки сервера
- wait_for:
    host: "{{ inventory_hostname }}"
    port: 22
    delay: 30              # ждать перед проверкой
    timeout: 600

# Дождаться порта
- wait_for:
    port: 8080
    delay: 5

# Дождаться рисунка в файле
- wait_for:
    path: /tmp/file.txt
    search_regex: "READY"
```

---

## Handlers

**Что это:**
- Вызываются через `notify`
- Выполняются один раз в конце
- Идеально для restart сервисов

**Пример:**
```yaml
tasks:
  - name: Copy config
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: restart nginx

handlers:
  - name: restart nginx
    systemd:
      name: nginx
      state: restarted
```

---

## Несколько Handlers

```yaml
tasks:
  - name: Copy Nginx config
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: 
      - validate nginx
      - restart nginx

  - name: Copy PHP config
    copy:
      src: php.ini
      dest: /etc/php/7.4/fpm/php.ini
    notify: restart php

handlers:
  - name: validate nginx
    shell: "nginx -t"
  
  - name: restart nginx
    systemd:
      name: nginx
      state: restarted
  
  - name: restart php
    systemd:
      name: php7.4-fpm
      state: restarted
```

---

## Практический Пример

```yaml
---
- name: Deploy and restart services
  hosts: webservers
  become: yes
  
  tasks:
    - name: Update packages
      apt:
        update_cache: yes
      async: 300
      poll: 10
    
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    
    - name: Copy main config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: restart nginx
    
    - name: Copy site config
      copy:
        src: site.conf
        dest: /etc/nginx/sites-available/site
      notify:
        - validate nginx
        - restart nginx
    
    - name: Enable site
      file:
        src: /etc/nginx/sites-available/site
        dest: /etc/nginx/sites-enabled/site
        state: link
      notify: restart nginx
    
    - name: Update application
      git:
        repo: https://github.com/user/app.git
        dest: /var/www/app
      async: 600
      poll: 30
    
    - name: Restart system
      reboot:
        reboot_timeout: 600
      async: 900
      poll: 0
    
    - name: Wait for system
      wait_for:
        host: "{{ inventory_hostname }}"
        port: 22
        delay: 30
        timeout: 300
      delegate_to: localhost
  
  handlers:
    - name: validate nginx
      shell: "nginx -t"
      listen: "validate nginx"
    
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
      listen: "restart nginx"
```

---

## Best Practices

✓ **Используй async для долгих операций**
```yaml
- apt: upgrade=full
  async: 1200
  poll: 60
```

✓ **Используй handlers для restart**
```yaml
tasks:
  - copy: src=config dest=/etc/config
    notify: restart service

handlers:
  - name: restart service
    systemd: name=service state=restarted
```

✓ **Используй wait_for после перезагрузки**
```yaml
- reboot:
- wait_for: port=22
```

✓ **Даже если notify вызван 5 раз, handler выполнится один раз**

---

**Следующее:** [[05-docker-project|Практический Проект: Docker Setup]]
