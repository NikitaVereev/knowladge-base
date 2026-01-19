---
title: 03 Отладка и Обработка Ошибок
---

---

## Debug Модуль

```yaml
# Вывести переменную
- debug:
    msg: "App port: {{ app_port }}"

# Вывести весь результат
- debug:
    var: my_variable

# Вывести факт
- debug:
    msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

---

## Register и Conditional

```yaml
- shell: "ps aux | grep nginx"
  register: ps_result

- debug:
    msg: "Nginx running"
  when: "'nginx' in ps_result.stdout"

- debug:
    msg: "Return code: {{ ps_result.rc }}"
```

---

## When Условия

```yaml
- debug: msg="Equal"
  when: app_port == 8080

- debug: msg="Not equal"
  when: app_port != 80

- debug: msg="Greater"
  when: ansible_processor_cores > 2

- debug: msg="In list"
  when: ansible_distribution in ['Ubuntu', 'Debian']

- debug: msg="Multiple conditions"
  when: 
    - ansible_distribution == "Ubuntu"
    - ansible_processor_cores >= 2
```

---

## Debugger (Breakpoints)

```yaml
- name: Task 1
  debug: msg="Before debug point"

- debugger: always
  name: Stop here to inspect

- name: Task 2
  shell: "important_command"
```

**Команды в debugger:**
- `p var` — вывести переменную
- `p *` — все переменные
- `c` — continue
- `q` — quit
- `r` — retry

**Опции:**
- `always` — всегда
- `on_failed` — при ошибке
- `never` — отключить

---

## Блоки (Blocks)

```yaml
- block:
    - name: Task 1
      apt: name=nginx state=present
    
    - name: Task 2
      systemd: name=nginx state=started
  
  become: yes                # применяется ко всем
  when: ansible_distribution == "Ubuntu"
```

---

## Rescue и Always

```
ПОПЫТКА → ОШИБКА? → RESCUE → ALWAYS → КОНЕЦ
   ↓                  ↓        ↓
 success             handle   cleanup
```

**Пример:**
```yaml
- block:
    - name: Запустить сервис
      systemd:
        name: nginx
        state: started
  
  rescue:
    - name: Показать ошибку
      debug:
        msg: "Error: {{ ansible_failed_result }}"
    
    - name: Установить если нет
      apt:
        name: nginx
        state: present
  
  always:
    - name: Проверить статус
      shell: "systemctl is-active nginx"
      ignore_errors: yes
```

---

## Управление Ошибками

**Игнорировать ошибку:**
```yaml
- shell: "exit 1"
  ignore_errors: yes
```

**Определить ошибку:**
```yaml
- shell: "curl http://localhost"
  register: result
  failed_when: 
    - result.rc != 0
    - "'Connection refused' in result.stderr"
```

**Когда не changed:**
```yaml
- shell: "echo hello"
  changed_when: False
```

**Остановить playbook:**
```yaml
- shell: "critical_command"
  any_errors_fatal: yes
```

---

## Практический Пример

```yaml
---
- name: Setup with error handling
  hosts: webservers
  become: yes
  
  tasks:
    - block:
        - name: Update package cache
          apt:
            update_cache: yes
        
        - name: Install packages
          apt:
            name:
              - nginx
              - curl
            state: present
        
        - name: Check Nginx config
          shell: "nginx -t"
          register: nginx_test
      
      rescue:
        - name: Log error
          debug:
            msg: "Setup failed: {{ ansible_failed_result }}"
        
        - name: Rollback
          apt:
            name: nginx
            state: absent
      
      always:
        - name: Final status
          debug:
            msg: "Task completed"
```

---

## Комбинированные Условия

```yaml
- debug: msg="Complex condition"
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_processor_cores >= 2
    - ansible_memtotal_mb > 1024
    - ansible_os_family not in ['Windows']
```

---

## Проверка Перед Выполнением

```bash
# Dry-run (check mode)
ansible-playbook playbook.yml --check

# Dry-run + verbose
ansible-playbook playbook.yml --check -vvv

# Diff mode (показать изменения)
ansible-playbook playbook.yml --diff
```

---

**Следующее:** [[04-async-handlers|Асинхронные Задачи и Handlers]]
