---
title: "Написание и запуск Playbooks"
type: how-to
tags: [ansible, playbook, ad-hoc, loops, when, handlers, tags, check-mode, limit, verbosity]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/index.html"
related:
  - "[[ansible/explanation/playbook-anatomy]]"
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/how-to/use-templates]]"
  - "[[ansible/how-to/debug-playbooks]]"
---

# Написание и запуск Playbooks

> **TL;DR:** `ansible-playbook site.yml` — запуск. `--check` — dry run. `--limit` — часть хостов.
> `loop` — повторение, `when` — условия, `notify` — handlers, `tags` — выборочный запуск.

## Запуск Playbook

```bash
# Базовый запуск
ansible-playbook -i inventory/hosts.yml site.yml

# Если inventory указан в ansible.cfg — без -i
ansible-playbook site.yml
```

### Ключевые флаги

```bash
# Dry run (без изменений)
ansible-playbook site.yml --check

# Показать diff файлов (что изменилось)
ansible-playbook site.yml --diff

# Check + Diff = идеальная проверка перед деплоем
ansible-playbook site.yml --check --diff

# Ограничить хосты
ansible-playbook site.yml --limit web-01
ansible-playbook site.yml --limit "webservers:!web-03"
ansible-playbook site.yml --limit "webservers:&production"   # пересечение групп

# Теги
ansible-playbook site.yml --tags "deploy,config"
ansible-playbook site.yml --skip-tags "debug"

# Extra vars (наивысший приоритет)
ansible-playbook site.yml -e "app_version=2.1.0 force=true"

# Verbosity (отладка)
ansible-playbook site.yml -v      # результаты задач
ansible-playbook site.yml -vv     # входные параметры модулей
ansible-playbook site.yml -vvv    # SSH-соединения

# Запросить sudo-пароль
ansible-playbook site.yml -K

# Vault-пароль
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass

# Step-by-step (подтверждение каждой задачи)
ansible-playbook site.yml --step

# Начать с конкретной задачи
ansible-playbook site.yml --start-at-task "Deploy application"
```

## Ad-Hoc команды

Выполнить одну задачу без playbook — удобно для быстрых проверок.

```bash
# Синтаксис
ansible [группа] -m [модуль] -a "[аргументы]"

# Ping (проверка SSH + Python)
ansible all -m ping

# Shell-команда
ansible webservers -m shell -a "uptime"
ansible all -m shell -a "df -h"

# Копировать файл
ansible webservers -m copy -a "src=./app.conf dest=/tmp/"

# Установить пакет (с sudo)
ansible webservers -m apt -a "name=htop state=present" -b

# Собрать факты
ansible web-01 -m setup
ansible web-01 -m setup -a "filter=ansible_distribution*"

# Перезапустить сервис
ansible webservers -m service -a "name=nginx state=restarted" -b
```

## Циклы (loop)

### Простой список

```yaml
- name: Установить пакеты
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - vim
    - htop
```

Оптимизация — передать список напрямую (модуль `apt` это поддерживает):

```yaml
- name: Установить пакеты (быстрее)
  ansible.builtin.apt:
    name:
      - git
      - curl
      - vim
      - htop
    state: present
```

### Список словарей

```yaml
- name: Создать пользователей
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: 'alice', groups: 'admin' }
    - { name: 'bob', groups: 'dev' }
    - { name: 'charlie', groups: 'dev' }
```

### Цикл по словарю (dict2items)

```yaml
vars:
  sysctl_settings:
    net.ipv4.ip_forward: 1
    vm.swappiness: 10

tasks:
  - name: Apply sysctl settings
    ansible.posix.sysctl:
      name: "{{ item.key }}"
      value: "{{ item.value }}"
    loop: "{{ sysctl_settings | dict2items }}"
```

### Register в циклах

```yaml
- name: Проверить сервисы
  ansible.builtin.command: "systemctl status {{ item }}"
  loop:
    - nginx
    - postgresql
  register: service_status
  ignore_errors: yes

# service_status.results — список результатов
- name: Показать упавшие
  ansible.builtin.debug:
    msg: "{{ item.item }} is down"
  loop: "{{ service_status.results }}"
  when: item.rc != 0
```

## Условия (when)

### Базовые примеры

```yaml
# По типу ОС
- name: Install (Debian)
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

- name: Install (RHEL)
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"

# По переменной
- name: Run migrations
  ansible.builtin.command: python manage.py migrate
  when: run_migrations | default(false) | bool
```

### С register

```yaml
- name: Проверить наличие конфига
  ansible.builtin.stat:
    path: /etc/app/config.ini
  register: config_file

- name: Создать дефолтный конфиг
  ansible.builtin.template:
    src: config.ini.j2
    dest: /etc/app/config.ini
  when: not config_file.stat.exists
```

### Сложные условия

```yaml
# AND (список = все условия должны быть истинны)
when:
  - ansible_distribution == "Ubuntu"
  - ansible_distribution_version is version('22.04', '>=')
  - not config_file.stat.exists

# OR
when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

# Переменная определена / не определена
when: my_var is defined
when: my_var is undefined
```

## Handlers (обработчики)

```yaml
tasks:
  - name: Обновить конфиг Nginx
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart Nginx            # ← триггер

  - name: Обновить конфиг сайта
    ansible.builtin.template:
      src: site.conf.j2
      dest: /etc/nginx/conf.d/site.conf
    notify: Restart Nginx            # ← тот же триггер

handlers:
  - name: Restart Nginx              # ← выполнится ОДИН РАЗ в конце
    ansible.builtin.service:
      name: nginx
      state: restarted
```

**Правила:**
1. Handler запускается только при `changed` в notify-задаче
2. Выполняется **один раз** в конце Play (даже при множественных notify)
3. Принудительный вызов раньше: `- meta: flush_handlers`

### Listen (группировка handlers)

```yaml
handlers:
  - name: Restart Nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
    listen: "restart web stack"

  - name: Restart PHP-FPM
    ansible.builtin.service:
      name: php-fpm
      state: restarted
    listen: "restart web stack"

tasks:
  - name: Update config
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/app.conf
    notify: "restart web stack"      # ← запустит ОБА handler'а
```

## Теги (tags)

```yaml
tasks:
  - name: Install packages
    ansible.builtin.apt:
      name: nginx
    tags: [install, setup]

  - name: Configure
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags: [config, nginx]

  - name: Destroy everything
    ansible.builtin.file:
      path: /opt/app
      state: absent
    tags: [never, destroy]           # never = не запустится без явного --tags
```

```bash
# Только установка
ansible-playbook site.yml --tags install

# Всё кроме конфигурации
ansible-playbook site.yml --skip-tags config

# Явный запуск «опасной» задачи
ansible-playbook site.yml --tags destroy

# Посмотреть доступные теги
ansible-playbook site.yml --list-tags
```

Специальные теги:
- `always` — выполнится всегда (если не `--skip-tags always`)
- `never` — не выполнится никогда (если не `--tags never` или `--tags destroy`)

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `loop` для модулей, принимающих список | Медленно (N SSH-операций вместо 1) | `apt: name: [git, curl, vim]` вместо `loop` |
| `when: variable` вместо `when: variable \| bool` | Строка "false" = truthy | Всегда `\| bool` для строковых переменных |
| Handler не срабатывает | Notify-задача вернула `ok` (не `changed`) | Проверить: файл действительно изменился? |
| `--check` не работает с shell/command | `skipping: conditional check mode` | Добавить `check_mode: no` к задачам, от которых зависят другие |
| Теги на include, а не на задачах | Задачи внутри include не наследуют теги | Используй `import_tasks` (статический) для наследования тегов |