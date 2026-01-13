---
title: 01 Структура Playbooks
---

---

## Что такое Playbook

```yaml
---                           # YAML начало
- name: Имя playbook'а
  hosts: webservers          # На каких хостах выполнять
  become: yes                # Использовать sudo
  gather_facts: yes          # Собрать информацию о хостах
  
  vars:                      # Переменные
    app_port: 8080
  
  pre_tasks:                 # Выполняются первыми
    - debug: msg="Starting"
  
  tasks:                     # Основные задачи
    - apt: name=nginx state=present
  
  post_tasks:                # Выполняются последними
    - debug: msg="Done!"
  
  handlers:                  # Вызываются при notify
    - name: restart nginx
      systemd: name=nginx state=restarted
```

---

## Структура Play

```
┌─────────────────────────────────────────┐
│          PLAYBOOK (site.yml)            │
├─────────────────────────────────────────┤
│ PLAY 1: Configure webservers            │
│  ├─ hosts: webservers                   │
│  ├─ vars: {...}                         │
│  └─ tasks:                              │
│     ├─ pre_tasks                        │
│     ├─ Task 1: Install nginx            │
│     ├─ Task 2: Copy config              │
│     └─ post_tasks                       │
├─────────────────────────────────────────┤
│ PLAY 2: Configure databases             │
│  ├─ hosts: databases                    │
│  └─ tasks: {...}                        │
└─────────────────────────────────────────┘
```

---

## Жизненный Цикл Playbook

```
1. ПАРСИНГ
   └─ Чтение playbook.yml
   └─ Проверка YAML синтаксиса

2. ПОДГОТОВКА
   └─ Подключение к каждому хосту (SSH)
   └─ Сбор фактов (gather_facts)

3. ВЫПОЛНЕНИЕ
   └─ pre_tasks
   └─ tasks:
      └─ Для каждой task:
         ├─ Проверить when условие
         ├─ Запустить модуль
         ├─ Получить результат
         └─ Если changed → notify handler
   └─ post_tasks
   └─ Handlers (один раз в конце)

4. ОТЧЕТ
   └─ Вывести результаты
   └─ Показать изменения
```

---

## Простой Пример

```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: yes
  
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    
    - name: Start Nginx
      systemd:
        name: nginx
        state: started
        enabled: yes
```

**Выполнение:**
```bash
ansible-playbook -i hosts.ini playbook.yml
```

---

## Запуск Playbook

```bash
# Обычный запуск
ansible-playbook -i hosts.ini playbook.yml

# Dry-run (проверка без изменений)
ansible-playbook -i hosts.ini playbook.yml --check

# С подробным выводом
ansible-playbook -i hosts.ini playbook.yml -vvv

# На конкретной группе
ansible-playbook -i hosts.ini playbook.yml -l webservers

# С запросом пароля sudo
ansible-playbook -i hosts.ini playbook.yml -K

# Останавливаться на ошибке
ansible-playbook -i hosts.ini playbook.yml -e "ansible_verbosity=3"
```

---

## Handlers (Обработчики)

**Что это:**
- Вызываются только при `notify`
- Выполняются один раз в конце
- Используются для restart сервисов

**Пример:**
```yaml
tasks:
  - name: Copy Nginx config
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

**Даже если notify вызван 5 раз, handler выполнится один раз!**

---

## Pre_tasks и Post_tasks

```yaml
---
- name: Deploy application
  hosts: webservers
  
  pre_tasks:
    - name: Проверить connectivity
      ping:
    
    - name: Backup текущей версии
      shell: "cp -r /opt/app /opt/app.backup"
  
  tasks:
    - name: Deploy новая версия
      git:
        repo: https://github.com/user/app.git
        dest: /opt/app
  
  post_tasks:
    - name: Проверить приложение
      shell: "curl http://localhost:8080/health"
    
    - name: Отправить уведомление
      debug:
        msg: "Deploy completed!"
```

---

## Практический Пример

```yaml
---
- name: Setup Nginx web server
  hosts: webservers
  become: yes
  
  vars:
    nginx_port: 80
    nginx_user: www-data
  
  pre_tasks:
    - name: Update package cache
      apt:
        update_cache: yes
  
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    
    - name: Copy Nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: restart nginx
    
    - name: Create web directory
      file:
        path: /var/www/html
        state: directory
        owner: "{{ nginx_user }}"
        group: "{{ nginx_user }}"
    
    - name: Copy index.html
      copy:
        content: "<h1>Hello from {{ ansible_hostname }}</h1>"
        dest: /var/www/html/index.html
    
    - name: Enable Nginx
      systemd:
        name: nginx
        enabled: yes
        state: started
  
  post_tasks:
    - name: Verify Nginx is running
      shell: "curl http://localhost"
      register: curl_result
      failed_when: curl_result.rc != 0
  
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

---

## Параметры Playbook

| Параметр | Описание |
|----------|---------|
| `hosts` | Группа из инвентаря (webservers, all, localhost) |
| `name` | Название play (для логов) |
| `become` | Использовать sudo (yes/no) |
| `become_user` | Пользователь для sudo (по умолчанию root) |
| `gather_facts` | Собирать факты о системе (yes/no) |
| `vars` | Переменные для этого play |
| `tasks` | Список задач |
| `handlers` | Обработчики |
| `strategy` | Способ выполнения (linear, free, debug) |

---

## Проблемы и Решения

**"undefined variable" ошибка**
```yaml
# Проверить переменную определена
- debug:
    msg: "{{ undefined_var | default('default_value') }}"
```

**Task не выполняется**
```yaml
# Проверить when условие
- debug:
    msg: "Running"
  when: ansible_distribution == "Ubuntu"
```

**Handlers не выполняются**
```yaml
# Убедиться в точном названии
tasks:
  - copy: src=file dest=/tmp
    notify: restart service   # Точное имя!

handlers:
  - name: restart service    # Должно совпадать!
    systemd: name=service state=restarted
```

---

**Следующее:** [[02-variables-facts|Переменные и Facts]]
