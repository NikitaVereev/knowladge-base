---
title: 02 Переменные и Facts
---

---

## Переменные: Синтаксис

```yaml
vars:
  app_name: myapp
  app_port: 8080
  app_version: "2.0"
  nginx_workers: 4
  debug_mode: true

tasks:
  - debug:
      msg: "App {{ app_name }} on port {{ app_port }}"
```

---

## Источники Переменных

**В playbook:**
```yaml
- hosts: webservers
  vars:
    app_env: production
```

**В task:**
```yaml
- apt: name=nginx
  vars:
    my_var: value
```

**Из файла:**
```yaml
- hosts: webservers
  vars_files:
    - vars/common.yml
    - vars/{{ ansible_distribution }}.yml
```

**Из инвентаря:**
```ini
[webservers]
web1 app_port=8080 app_name=myapp
```

**Command line:**
```bash
ansible-playbook playbook.yml -e "app_port=9000"
ansible-playbook playbook.yml -e @vars.json
```

---

## Приоритеты (High to Low)

```
1. extra-vars (-e)                    ↑ Наивысший
2. set_fact / include_vars
3. Task/Block vars
4. Play vars
5. Inventory host vars
6. Inventory group vars
7. Default values                     ↓ Самый низкий
```

**Пример:**
```bash
# Это переопределит всё
ansible-playbook playbook.yml -e "app_port=9999"
```

---

## Facts (Автоматические Переменные)

```
Ansible автоматически собирает информацию о каждом хосте
```

**Основные facts:**
```yaml
{{ ansible_distribution }}        # Ubuntu, CentOS, Debian
{{ ansible_distribution_version }} # 24.04, 8.5
{{ ansible_os_family }}            # Debian, RedHat
{{ ansible_hostname }}             # имя хоста
{{ ansible_fqdn }}                 # полное имя
{{ ansible_processor_cores }}      # кол-во ядер
{{ ansible_memtotal_mb }}          # всего памяти
{{ ansible_default_ipv4.address }} # IP адрес
```

**Использование в условиях:**
```yaml
- apt: name=curl
  when: ansible_distribution == "Ubuntu"

- debug:
    msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

**Отключить сбор фактов (ускорит выполнение):**
```yaml
- hosts: webservers
  gather_facts: no
```

---

## Register (Сохранение Результатов)

```yaml
- shell: "ps aux | grep nginx"
  register: ps_result

- debug:
    msg: "Nginx running"
  when: "'nginx' in ps_result.stdout"

- debug:
    var: ps_result  # Вывести весь результат
```

**Структура результата:**
```json
{
  "stdout": "output text",
  "stderr": "error text",
  "rc": 0,                    # return code
  "changed": true,
  "cmd": "command"
}
```

---

## When Условия

```yaml
- debug: msg="Ubuntu"
  when: ansible_distribution == "Ubuntu"

- debug: msg="Multiple cores"
  when: ansible_processor_cores > 2

- debug: msg="CentOS or RHEL"
  when: ansible_os_family in ['RedHat']

- debug: msg="File exists"
  when: ansible_stat.stat.exists

- debug: msg="Port open"
  when: "'8080' in netstat_output.stdout"
```

---

## Практический Пример

```yaml
---
- name: Configure web server
  hosts: webservers
  
  vars:
    app_name: myapp
    app_port: 8080
    app_env: production
  
  tasks:
    - name: Gather facts
      setup:
    
    - name: Install packages
      apt:
        name:
          - nginx
          - curl
        state: present
      when: ansible_distribution == "Ubuntu"
      register: install_result
    
    - name: Show install result
      debug:
        msg: "{{ app_name }} installed on {{ ansible_hostname }}"
      when: install_result is changed
    
    - name: Check port availability
      shell: "netstat -tuln | grep {{ app_port }}"
      register: port_check
      ignore_errors: yes
    
    - debug:
        msg: "Port {{ app_port }} is available"
      when: port_check.rc == 0
    
    - name: Show system info
      debug:
        msg: |
          Host: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Cores: {{ ansible_processor_cores }}
          Memory: {{ ansible_memtotal_mb }} MB
          IP: {{ ansible_default_ipv4.address }}
```

---

## Шаблоны Jinja2

```yaml
- name: Copy config with variables
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

**nginx.conf.j2:**
```
server {
    listen {{ app_port }};
    server_name {{ ansible_hostname }};
    
    location / {
        proxy_pass http://localhost:3000;
    }
    
    {% if app_env == "production" %}
    ssl_protocols TLSv1.2 TLSv1.3;
    {% endif %}
}
```

---

## Условные Фильтры

```yaml
- debug:
    msg: "{{ app_port | default(80) }}"  # default значение

- debug:
    msg: "{{ app_name | upper }}"        # uppercase

- debug:
    msg: "{{ list_var | join(',') }}"    # join

- debug:
    msg: "{{ dict_var | to_nice_json }}" # to JSON

- set_fact:
    result: "{{ input | bool }}"         # convert to bool
```

---

**Следующее:** [[03-debug-blocks|Отладка и Обработка Ошибок]]
