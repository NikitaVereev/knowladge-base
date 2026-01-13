---
title: 04 Модули и Выполнение
---

---

## Встроенные Модули

**Пакеты:**
```yaml
# Debian/Ubuntu
- apt: name=nginx state=present

# CentOS/RHEL 8+
- dnf: name=nginx state=present

# CentOS/RHEL 6-7
- yum: name=nginx state=present
```

**Сервисы:**
```yaml
# Современный способ (systemd)
- systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes

# Старый способ
- service: name=nginx state=restarted
```

**Файлы:**
```yaml
# Копировать файл
- copy:
    src: /local/nginx.conf
    dest: /etc/nginx/nginx.conf
    mode: '0644'

# Обработать Jinja2 шаблон
- template:
    src: templates/app.conf.j2
    dest: /etc/app.conf

# Создать директорию
- file:
    path: /opt/myapp
    state: directory
    mode: '0755'
    owner: app
    group: app
```

**Команды:**
```yaml
# Shell команда (можно использовать трубы, переменные, и т.д.)
- shell: "ps aux | grep nginx"
  register: ps_output

# Простая команда (без трубов)
- command: /usr/bin/whoami

# Выполнить скрипт
- script: /local/deploy.sh
```

**Пользователи:**
```yaml
# Создать пользователя
- user:
    name: webapp
    state: present
    shell: /bin/bash
    groups: www-data
    createhome: yes

# Добавить SSH ключ
- authorized_key:
    user: ubuntu
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
```

**Загрузки:**
```yaml
# Скачать файл
- get_url:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz

# Клонировать репозиторий
- git:
    repo: https://github.com/user/repo.git
    dest: /opt/repo
    version: main

# Распаковать архив
- unarchive:
    src: /tmp/file.tar.gz
    dest: /opt/
```

---

## Документация по Модулям

```bash
# Просмотреть документацию
ansible-doc apt

# Краткая справка
ansible-doc -s apt

# Список всех модулей
ansible-doc -l

# Примеры
ansible-doc -e apt
```

---

## Ad-Hoc Команды

**Синтаксис:**
```bash
ansible <hosts> -i <inventory> -m <module> -a <arguments>
```

---

## Примеры Ad-Hoc

**Пинг все хосты:**
```bash
ansible all -i hosts.ini -m ping
```

**Получить информацию о системе:**
```bash
ansible webservers -i hosts.ini -m setup | grep ansible_distribution

# Отфильтровать конкретный факт
ansible webservers -i hosts.ini -m setup -a "filter=ansible_processor_cores"
```

**Выполнить команду:**
```bash
ansible webservers -i hosts.ini -m shell -a "uptime"

ansible all -i hosts.ini -m shell -a "ps aux | grep nginx"
```

**Установить пакет:**
```bash
# На Ubuntu/Debian
ansible webservers -i hosts.ini -m apt -a "name=curl state=present" -b

# На CentOS/RHEL
ansible webservers -i hosts.ini -m dnf -a "name=curl state=present" -b
```

**Управлять сервисом:**
```bash
# Запустить
ansible webservers -i hosts.ini -m systemd -a "name=nginx state=started" -b

# Перезагрузить
ansible webservers -i hosts.ini -m systemd -a "name=nginx state=restarted" -b

# Остановить
ansible webservers -i hosts.ini -m systemd -a "name=nginx state=stopped" -b
```

**Создать пользователя:**
```bash
ansible all -i hosts.ini -m user -a "name=testuser state=present" -b
```

**Скопировать файл:**
```bash
ansible webservers -i hosts.ini -m copy -a "src=/local/file dest=/tmp/file" -b
```

---

## Параметры Ad-Hoc Команд

| Параметр | Описание |
|----------|---------|
| `-i` | Инвентарь (hosts.ini) |
| `-m` | Модуль (apt, shell, systemd) |
| `-a` | Аргументы модуля |
| `-b` | Выполнить с `become` (sudo) |
| `-u` | SSH пользователь |
| `-k` | Запросить SSH пароль |
| `-K` | Запросить пароль sudo |
| `-v`, `-vv`, `-vvv` | Подробный вывод |
| `-l` | Ограничить хосты (web1,web2) |
| `--check` | Dry-run (без изменений) |

**Примеры:**
```bash
# С sudo и вводом пароля
ansible webservers -i hosts.ini -m apt -a "update_cache=yes" -b -K

# На конкретные хосты
ansible all -i hosts.ini -m ping -l "web1,web2"

# Подробный вывод
ansible webservers -i hosts.ini -m setup -vvv

# Dry-run
ansible webservers -i hosts.ini -m apt -a "name=nginx" -b --check
```

---

## Register и Conditional

**Сохранить результат:**
```bash
ansible webservers -i hosts.ini -m shell -a "ps aux | grep nginx" \
  -b | tee nginx_status.log
```

**Использовать в playbook:**
```yaml
- shell: "curl http://localhost:8080"
  register: curl_result

- debug:
    msg: "Service is running"
  when: curl_result.rc == 0
```

---

## Когда Использовать

**Ad-hoc команды (быстро):**
```bash
# ✓ Быстрая проверка
ansible all -i hosts.ini -m ping

# ✓ Срочное исправление
ansible webservers -i hosts.ini -m systemd -a "name=nginx state=restarted" -b
```

**Playbooks (переиспользуемо):**
```yaml
# ✓ Сохраняется в Git
# ✓ Код review
# ✓ История изменений
- systemd: name=nginx state=restarted
```

---

**Следующее:** [[05-best-practices|Best Practices и Примеры]]
