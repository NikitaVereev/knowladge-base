---
title: 05 Best Practices и Примеры
---

---

## Структура Проекта

```
ansible-project/
├── hosts.yaml              # Инвентарь
├── ansible.cfg             # Конфигурация
├── playbooks/              # Playbooks
│   ├── site.yml           # Main playbook
│   ├── webservers.yml
│   ├── databases.yml
│   └── deploy.yml
├── roles/                  # Переиспользуемые роли
│   ├── common/
│   ├── webserver/
│   └── database/
├── group_vars/             # Переменные групп
│   ├── all.yml
│   ├── webservers.yml
│   └── databases.yml
├── host_vars/              # Переменные хостов
│   ├── web1.yml
│   └── db1.yml
├── templates/              # Jinja2 шаблоны
│   ├── nginx.conf.j2
│   └── app.conf.j2
├── files/                  # Статические файлы
│   └── ssh_config
└── README.md
```

---

## Именование Переменных

**✓ Хорошо:**
```yaml
app_name: myapp
app_port: 8080
app_version: "2.0"
http_timeout: 30
nginx_workers: 4
```

**✗ Плохо:**
```yaml
name: myapp           # слишком общее
port: 8080           # конфликты между переменными
v: "2.0"             # не понятное название
timeout_http: 30     # неконсистентное
workers: 4           # что это за workers?
```

---

## Документирование

**README.md:**
```markdown
# Ansible Configuration

## Что это

Автоматизация настройки инфраструктуры на примере веб-приложения.

## Структура

- `hosts.yaml` — инвентарь (веб-серверы, БД)
- `playbooks/` — основные сценарии
- `roles/` — переиспользуемые роли

## Использование

```bash
# Dry-run перед применением
ansible-playbook playbooks/site.yml --check

# Применить конфигурацию
ansible-playbook playbooks/site.yml

# На конкретной группе
ansible-playbook playbooks/deploy.yml -l webservers
```

## Требования

- Ansible 2.10+
- Python 3.9+ на целевых серверах
- SSH доступ до хостов


---

## Пример 1: Установка Nginx + PHP + MySQL

```yaml
---
- name: Setup LEMP Stack
  hosts: webservers
  become: yes
  
  vars:
    app_name: myapp
    app_port: 8080
    php_version: "8.2"
    mysql_version: "8.0"
  
  tasks:
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    
    - name: Install PHP and extensions
      apt:
        name:
          - "php{{ php_version }}"
          - "php{{ php_version }}-fpm"
          - "php{{ php_version }}-mysql"
          - "php{{ php_version }}-curl"
          - "php{{ php_version }}-json"
        state: present
    
    - name: Install MySQL client
      apt:
        name: mysql-client
        state: present
    
    - name: Copy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
      notify: restart nginx
    
    - name: Enable Nginx site
      file:
        src: /etc/nginx/sites-available/{{ app_name }}
        dest: /etc/nginx/sites-enabled/{{ app_name }}
        state: link
    
    - name: Start services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - nginx
        - "php{{ php_version }}-fpm"
  
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

---

## Пример 2: Deploy Node.js Приложения

```yaml
---
- name: Deploy Node.js App
  hosts: webservers
  
  vars:
    app_repo: https://github.com/user/myapp.git
    app_branch: main
    app_path: /opt/myapp
    node_env: production
  
  tasks:
    - name: Install dependencies
      become: yes
      apt:
        name:
          - nodejs
          - npm
          - git
        state: present
    
    - name: Create app directory
      become: yes
      file:
        path: "{{ app_path }}"
        state: directory
        owner: app
        group: app
        mode: '0755'
    
    - name: Clone repository
      git:
        repo: "{{ app_repo }}"
        dest: "{{ app_path }}"
        version: "{{ app_branch }}"
    
    - name: Install npm dependencies
      shell: "cd {{ app_path }} && npm ci"
      environment:
        NODE_ENV: "{{ node_env }}"
    
    - name: Build application
      shell: "cd {{ app_path }} && npm run build"
      environment:
        NODE_ENV: "{{ node_env }}"
    
    - name: Copy systemd service
      become: yes
      template:
        src: templates/app.service.j2
        dest: /etc/systemd/system/myapp.service
      notify: restart app
    
    - name: Start application
      become: yes
      systemd:
        name: myapp
        state: started
        enabled: yes
        daemon_reload: yes
  
  handlers:
    - name: restart app
      become: yes
      systemd:
        name: myapp
        state: restarted
```

---

## Пример 3: Настройка Брандмауэра и SSH

```yaml
---
- name: Harden SSH and Firewall
  hosts: all
  become: yes
  
  tasks:
    - name: Update package cache
      apt:
        update_cache: yes
    
    - name: Install UFW
      apt:
        name: ufw
        state: present
    
    - name: Configure SSH
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - regexp: "^#Port "
          line: "Port 22"
        - regexp: "^#PermitRootLogin "
          line: "PermitRootLogin no"
        - regexp: "^#PasswordAuthentication "
          line: "PasswordAuthentication no"
      notify: restart ssh
    
    - name: Configure firewall
      community.general.ufw:
        rule: "{{ item.rule }}"
        port: "{{ item.port }}"
        proto: "{{ item.proto }}"
        state: enabled
      loop:
        - rule: allow
          port: "22"
          proto: tcp
        - rule: allow
          port: "80"
          proto: tcp
        - rule: allow
          port: "443"
          proto: tcp
    
    - name: Enable firewall
      community.general.ufw:
        state: enabled
  
  handlers:
    - name: restart ssh
      systemd:
        name: ssh
        state: restarted
```

---

## Пример 4: Регулярные Обновления

```yaml
---
- name: System Updates
  hosts: all
  become: yes
  
  tasks:
    - name: Update all packages
      apt:
        upgrade: full
        update_cache: yes
      register: update_result
    
    - name: Check if reboot needed
      stat:
        path: /var/run/reboot-required
      register: reboot_required
    
    - name: Reboot if needed
      reboot:
        msg: "System update requires reboot"
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 30
      when: reboot_required.stat.exists
    
    - name: Log updates
      copy:
        content: "Updated at: {{ ansible_date_time.iso8601 }}\n"
        dest: /var/log/ansible-updates.log
        append: yes
```

---

## Обработка Ошибок

**Проверка перед выполнением:**
```yaml
- name: Check if service is running
  systemd:
    name: nginx
    state: started
  register: service_status
  ignore_errors: yes

- debug:
    msg: "Service not running!"
  when: service_status is failed
```

**Откат при ошибках:**
```yaml
- block:
    - name: Deploy new version
      copy:
        src: /tmp/app.tar.gz
        dest: /opt/app.tar.gz
  
  rescue:
    - name: Rollback
      shell: "cd /opt && tar xzf app.backup.tar.gz"
```

---

## Лучшие Практики

✓ **Версионируй в Git**
```bash
git init
git add .
git commit -m "Initial Ansible configuration"
```

✓ **Используй --check перед запуском**
```bash
ansible-playbook playbooks/site.yml --check
```

✓ **Логируй все изменения**
```bash
ansible-playbook playbooks/site.yml -l all > deploy_$(date +%Y%m%d_%H%M%S).log
```

✓ **Структурируй код**
- Используй roles для переиспользуемого кода
- Разделяй playbooks по функциям
- Храни переменные в отдельных файлах

✓ **Документируй**
- Напиши README.md
- Добавь комментарии в playbooks
- Опиши переменные

---

