---
title: "03 — Проект с ролями"
type: tutorial
tags: [ansible, tutorial, roles, refactoring, defaults, meta, galaxy, structure]
sources:
  docs: "https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html"
related:
  - "[[ansible/tutorials/02-server-setup]]"
  - "[[ansible/tutorials/04-deploy-docker-app]]"
  - "[[ansible/how-to/create-roles]]"
  - "[[ansible/explanation/roles-and-collections]]"
---

# Tutorial 03 — Проект с ролями

> **Цель:** Рефакторинг монолитного playbook из Tutorial 02 в переиспользуемые роли.
> 3 роли: `common`, `hardening`, `docker`. Практика: defaults, handlers, meta, Galaxy requirements.

**Время:** ~40 минут
**Требования:** Пройден Tutorial 02.

## Шаг 1. Новая структура

```bash
mkdir -p roles-project/{roles/{common,hardening,docker}/{tasks,handlers,defaults,templates},inventory/group_vars/all}
cd roles-project
```

```
roles-project/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all/
│           ├── vars.yml
│           └── vault.yml
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   └── defaults/main.yml
│   ├── hardening/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── defaults/main.yml
│   │   └── templates/sshd_config.j2
│   └── docker/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── defaults/main.yml
└── site.yml
```

## Шаг 2. Роль `common` — базовые пакеты и пользователи

**roles/common/defaults/main.yml:**

```yaml
---
common_packages:
  - curl
  - wget
  - git
  - vim
  - htop
  - unzip

common_timezone: UTC

common_admin_user: deploy
common_admin_groups: sudo
common_admin_shell: /bin/bash
common_admin_ssh_key: ""              # переопределить в group_vars
```

**roles/common/tasks/main.yml:**

```yaml
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install base packages
  ansible.builtin.apt:
    name: "{{ common_packages }}"
    state: present

- name: Set timezone
  community.general.timezone:
    name: "{{ common_timezone }}"

- name: Create admin user
  ansible.builtin.user:
    name: "{{ common_admin_user }}"
    groups: "{{ common_admin_groups }}"
    shell: "{{ common_admin_shell }}"
    create_home: yes

- name: Add SSH key
  ansible.posix.authorized_key:
    user: "{{ common_admin_user }}"
    key: "{{ common_admin_ssh_key }}"
  when: common_admin_ssh_key | length > 0

- name: Allow passwordless sudo
  ansible.builtin.copy:
    content: "{{ common_admin_user }} ALL=(ALL) NOPASSWD:ALL"
    dest: "/etc/sudoers.d/{{ common_admin_user }}"
    mode: '0440'
    validate: "visudo -cf %s"
```

## Шаг 3. Роль `hardening` — SSH + Firewall

**roles/hardening/defaults/main.yml:**

```yaml
---
hardening_ssh_port: 22
hardening_ssh_permit_root: "no"
hardening_ssh_password_auth: "no"

hardening_ufw_allowed_ports:
  - { port: 22, proto: tcp }
  - { port: 80, proto: tcp }
  - { port: 443, proto: tcp }

hardening_install_fail2ban: true
hardening_enable_auto_updates: true
```

**roles/hardening/tasks/main.yml:**

```yaml
---
- name: Install security packages
  ansible.builtin.apt:
    name:
      - ufw
      - "{{ 'fail2ban' if hardening_install_fail2ban else [] }}"
      - "{{ 'unattended-upgrades' if hardening_enable_auto_updates else [] }}"
    state: present

# --- SSH ---
- name: Deploy SSH config
  ansible.builtin.template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    validate: "sshd -t -f %s"
    backup: yes
  notify: Restart SSH

# --- UFW ---
- name: Allow required ports
  community.general.ufw:
    rule: allow
    port: "{{ item.port | string }}"
    proto: "{{ item.proto }}"
  loop: "{{ hardening_ufw_allowed_ports }}"

- name: Enable UFW
  community.general.ufw:
    state: enabled
    policy: deny

# --- Auto Updates ---
- name: Configure auto-updates
  ansible.builtin.copy:
    content: |
      APT::Periodic::Update-Package-Lists "1";
      APT::Periodic::Unattended-Upgrade "1";
      APT::Periodic::AutocleanInterval "7";
    dest: /etc/apt/apt.conf.d/20auto-upgrades
  when: hardening_enable_auto_updates
```

**roles/hardening/handlers/main.yml:**

```yaml
---
- name: Restart SSH
  ansible.builtin.systemd:
    name: ssh
    state: restarted
```

**roles/hardening/templates/sshd_config.j2:**

```
# Managed by Ansible — DO NOT EDIT
Port {{ hardening_ssh_port }}
PermitRootLogin {{ hardening_ssh_permit_root }}
PasswordAuthentication {{ hardening_ssh_password_auth }}
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

## Шаг 4. Роль `docker` — установка Docker

**roles/docker/defaults/main.yml:**

```yaml
---
docker_users: []
docker_log_max_size: "50m"
docker_log_max_file: "3"
docker_edition: ce                     # ce | ee
```

**roles/docker/tasks/main.yml:**

```yaml
---
- name: Install prerequisites
  ansible.builtin.apt:
    name:
      - ca-certificates
      - gnupg
      - lsb-release
    state: present

- name: Add Docker GPG key
  ansible.builtin.get_url:
    url: "https://download.docker.com/linux/ubuntu/gpg"
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
  retries: 3
  delay: 5

- name: Determine architecture
  ansible.builtin.set_fact:
    docker_arch: >-
      {{ 'amd64' if ansible_architecture == 'x86_64'
         else 'arm64' if ansible_architecture == 'aarch64'
         else 'armhf' }}

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: >-
      deb [arch={{ docker_arch }} signed-by=/etc/apt/keyrings/docker.asc]
      https://download.docker.com/linux/ubuntu
      {{ ansible_distribution_release }} stable
    filename: docker

- name: Install Docker
  ansible.builtin.apt:
    name:
      - "docker-{{ docker_edition }}"
      - "docker-{{ docker_edition }}-cli"
      - containerd.io
      - docker-compose-plugin
      - docker-buildx-plugin
    state: present
    update_cache: yes

- name: Configure Docker daemon
  ansible.builtin.copy:
    content: |
      {
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "{{ docker_log_max_size }}",
          "max-file": "{{ docker_log_max_file }}"
        },
        "no-new-privileges": true,
        "live-restore": true,
        "userland-proxy": false
      }
    dest: /etc/docker/daemon.json
  notify: Restart Docker

- name: Add users to docker group
  ansible.builtin.user:
    name: "{{ item }}"
    groups: docker
    append: yes
  loop: "{{ docker_users }}"

- name: Start and enable Docker
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: yes
```

**roles/docker/handlers/main.yml:**

```yaml
---
- name: Restart Docker
  ansible.builtin.systemd:
    name: docker
    state: restarted
    daemon_reload: yes
```

## Шаг 5. Собираем playbook

**site.yml** — теперь чистый и читаемый:

```yaml
---
- name: Production Server Setup
  hosts: all
  become: yes

  roles:
    - common
    - hardening
    - docker
```

**inventory/group_vars/all/vars.yml** — переопределяем defaults:

```yaml
---
# Common
common_admin_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
common_admin_groups: sudo,docker

# Hardening
hardening_ufw_allowed_ports:
  - { port: 22, proto: tcp }
  - { port: 80, proto: tcp }
  - { port: 443, proto: tcp }
  - { port: 8080, proto: tcp }

# Docker
docker_users: ["deploy"]
```

## Шаг 6. Запуск и проверка

```bash
# Syntax check
ansible-playbook site.yml --syntax-check

# Запуск
ansible-playbook site.yml --ask-vault-pass

# Только роль docker (через теги, если добавили)
ansible-playbook site.yml --tags docker
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| defaults/main.yml | Параметры с низким приоритетом — легко переопределить из group_vars |
| handlers в роли | Автоматически доступны из tasks этой роли |
| templates в роли | Ansible ищет в `roles/<name>/templates/` автоматически |
| Разделение ответственности | Каждая роль — одна задача. Можно переиспользовать |
| site.yml | 7 строк вместо 150. Читаемый, декларативный |

## Сравнение: до и после

| Метрика | Tutorial 02 (монолит) | Tutorial 03 (роли) |
|---------|----------------------|-------------------|
| site.yml | ~150 строк | 7 строк |
| Переиспользуемость | Никакая | Каждую роль можно подключить отдельно |
| Настройка | Правка YAML в середине файла | Переопределение через group_vars |
| Тестирование | Целиком | Каждую роль отдельно (Molecule) |

## Что дальше

→ [[ansible/tutorials/04-deploy-docker-app]] — деплой приложения в Docker с помощью Ansible

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Переменные без префикса роли | `port` из роли nginx перебила `port` из роли app | Всегда `<role>_<param>`: `docker_log_max_size` |
| Всё в `vars/main.yml` | Невозможно переопределить из playbook | Настраиваемые параметры → `defaults/`, константы → `vars/` |
| Handler из другой роли | `ERROR! The requested handler 'Restart Nginx' was not found` | Handler доступен только внутри своей роли (или через `listen`) |
| Забыли `validate` для SSH | Сломанный конфиг → потеряли доступ | Всегда `validate: "sshd -t -f %s"` |