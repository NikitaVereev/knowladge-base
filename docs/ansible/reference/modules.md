---
title: "Справочник модулей"
type: reference
tags: [ansible, modules, apt, file, template, service, command, shell, user, git, uri, fqcn]
sources:
  docs: "https://docs.ansible.com/ansible/latest/collections/index.html"
related:
  - "[[ansible/how-to/write-playbooks]]"
  - "[[ansible/reference/cheatsheet]]"
  - "[[ansible/reference/jinja2-filters]]"
---

# Справочник модулей Ansible

> **Справочник:** Модули, используемые в 90% задач. FQCN (Fully Qualified Collection Name)
> рекомендуется с Ansible 2.10+. Ctrl+F для поиска.

## Управление пакетами

```yaml
# apt (Debian/Ubuntu)
- ansible.builtin.apt:
    name: nginx
    state: present           # present | absent | latest
    update_cache: yes
    cache_valid_time: 3600   # не обновлять кэш чаще раза в час

# apt — несколько пакетов
- ansible.builtin.apt:
    name:
      - nginx
      - postgresql
      - redis
    state: present

# dnf (RHEL 8+/Fedora)
- ansible.builtin.dnf:
    name: httpd
    state: latest

# pip
- ansible.builtin.pip:
    name: docker
    state: present
    virtualenv: /opt/app/venv        # установить в virtualenv
    virtualenv_command: python3 -m venv

# npm
- community.general.npm:
    name: pm2
    global: yes
```

## Файлы и директории

```yaml
# file — создать директорию
- ansible.builtin.file:
    path: /opt/app
    state: directory         # directory | file | link | absent | touch
    owner: deploy
    group: deploy
    mode: '0755'
    recurse: yes             # рекурсивно для directory

# file — символическая ссылка
- ansible.builtin.file:
    src: /opt/app/releases/v2
    dest: /opt/app/current
    state: link

# copy — статический файл
- ansible.builtin.copy:
    src: files/nginx.conf        # из roles/myrole/files/
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes                  # сохранить предыдущую версию

# copy — контент напрямую
- ansible.builtin.copy:
    content: |
      server {
        listen 80;
      }
    dest: /etc/nginx/conf.d/app.conf

# template — Jinja2 шаблон
- ansible.builtin.template:
    src: templates/app.conf.j2   # из roles/myrole/templates/
    dest: /etc/app/app.conf
    validate: "nginx -t -c %s"   # валидация перед применением
    backup: yes

# lineinfile — добавить/изменить строку
- ansible.builtin.lineinfile:
    path: /etc/hosts
    line: "192.168.1.10 myserver"
    state: present

# lineinfile — заменить строку по regex
- ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
  notify: Restart sshd

# blockinfile — вставить блок текста
- ansible.builtin.blockinfile:
    path: /etc/nginx/nginx.conf
    insertafter: "http {"
    block: |
      upstream app {
        server 127.0.0.1:3000;
      }

# unarchive — распаковать архив
- ansible.builtin.unarchive:
    src: app-v2.tar.gz
    dest: /opt/app/
    remote_src: yes              # архив уже на сервере (не копировать)

# stat — проверить существование файла
- ansible.builtin.stat:
    path: /etc/app/config.yml
  register: config_file
# Использование: when: config_file.stat.exists

# find — поиск файлов
- ansible.builtin.find:
    paths: /var/log
    patterns: "*.log"
    age: 7d                      # старше 7 дней
  register: old_logs
```

## Сервисы и системы

```yaml
# systemd — управление сервисом
- ansible.builtin.systemd:
    name: nginx
    state: started           # started | stopped | restarted | reloaded
    enabled: yes             # автозапуск при boot
    daemon_reload: yes       # если изменился unit-файл

# service (универсальный, для старых систем)
- ansible.builtin.service:
    name: nginx
    state: restarted

# reboot
- ansible.builtin.reboot:
    reboot_timeout: 300
    msg: "Rebooting for kernel update"

# cron
- ansible.builtin.cron:
    name: "Daily backup"
    minute: "0"
    hour: "3"
    job: "/opt/scripts/backup.sh >> /var/log/backup.log 2>&1"
    user: deploy

# sysctl
- ansible.posix.sysctl:
    name: net.ipv4.ip_forward
    value: '1'
    sysctl_set: yes
    reload: yes
```

## Команды

```yaml
# command — безопасный (без shell-фич)
- ansible.builtin.command:
    cmd: /opt/app/migrate.sh
    chdir: /opt/app              # рабочая директория
    creates: /opt/app/.migrated  # не выполнять если файл существует

# shell — поддержка пайпов, редиректов
- ansible.builtin.shell: |
    cd /opt/app && npm run build 2>&1 | tee /var/log/build.log
  args:
    executable: /bin/bash

# raw — без Python на целевом хосте (bootstrap)
- ansible.builtin.raw: apt-get install -y python3
  become: yes
```

## Пользователи и доступ

```yaml
# user
- ansible.builtin.user:
    name: deploy
    groups: docker,sudo
    shell: /bin/bash
    create_home: yes
    generate_ssh_key: yes

# authorized_key
- ansible.posix.authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

# group
- ansible.builtin.group:
    name: appgroup
    gid: 1001
```

## Сеть и HTTP

```yaml
# uri — HTTP-запрос (healthcheck, API)
- ansible.builtin.uri:
    url: http://localhost:8080/health
    method: GET
    status_code: 200
    body_format: json
  register: health
  until: health.status == 200
  retries: 10
  delay: 5

# get_url — скачать файл
- ansible.builtin.get_url:
    url: https://example.com/app-v2.tar.gz
    dest: /tmp/app.tar.gz
    checksum: sha256:abc123...

# wait_for — ждать порт/файл
- ansible.builtin.wait_for:
    port: 5432
    delay: 5
    timeout: 60

# hostname
- ansible.builtin.hostname:
    name: web-01.example.com
```

## Git и деплой

```yaml
# git
- ansible.builtin.git:
    repo: https://github.com/org/app.git
    dest: /opt/app
    version: main                # branch, tag или commit SHA
    force: yes                   # отбросить локальные изменения
    accept_hostkey: yes

# synchronize (rsync wrapper)
- ansible.posix.synchronize:
    src: /local/app/
    dest: /opt/app/
    delete: yes
    rsync_opts:
      - "--exclude=node_modules"
      - "--exclude=.git"
```

## Отладка и контроль

```yaml
# debug — вывод переменных
- ansible.builtin.debug:
    msg: "Version: {{ app_version }}"
- ansible.builtin.debug:
    var: ansible_default_ipv4

# assert — проверка условий
- ansible.builtin.assert:
    that:
      - ansible_memtotal_mb >= 1024
      - app_version is defined
    fail_msg: "Requirements not met"

# pause — ожидание ввода
- ansible.builtin.pause:
    prompt: "Press Enter to continue deployment"

# set_fact — создать переменную
- ansible.builtin.set_fact:
    full_url: "http://{{ ansible_host }}:{{ app_port }}"

# meta
- ansible.builtin.meta: flush_handlers    # выполнить handlers сейчас
- ansible.builtin.meta: end_play          # завершить play
- ansible.builtin.meta: clear_facts       # очистить кэш фактов
```