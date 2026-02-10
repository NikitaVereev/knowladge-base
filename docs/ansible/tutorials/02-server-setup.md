---
title: "02 — Настройка сервера"
type: tutorial
tags: [ansible, tutorial, server, hardening, ssh, ufw, docker, users, lemp]
sources:
  docs: "https://docs.ansible.com/ansible/latest/getting_started/index.html"
related:
  - "[[ansible/tutorials/01-first-playbook]]"
  - "[[ansible/tutorials/03-roles-project]]"
  - "[[ansible/how-to/manage-secrets]]"
  - "[[ansible/how-to/write-playbooks]]"
---

# Tutorial 02 — Настройка сервера

> **Цель:** Production-ready настройка Ubuntu-сервера одним playbook: пользователи,
> SSH hardening, firewall (UFW), Docker, автообновления. Практика: vars, loops, handlers, when, vault.

**Время:** ~45 минут
**Требования:** Пройден Tutorial 01. Сервер Ubuntu 22.04+ с root/sudo SSH-доступом.

## Шаг 1. Структура проекта

```bash
mkdir -p server-setup/{group_vars/all,templates}
cd server-setup
```

```
server-setup/
├── ansible.cfg
├── inventory.yml
├── group_vars/
│   └── all/
│       ├── vars.yml          # открытые переменные
│       └── vault.yml         # зашифрованные секреты
├── templates/
│   └── sshd_config.j2
└── site.yml
```

## Шаг 2. Конфигурация и inventory

**ansible.cfg:**

```ini
[defaults]
inventory = ./inventory.yml
stdout_callback = yaml
retry_files_enabled = False
host_key_checking = False

[ssh_connection]
pipelining = True
```

**inventory.yml:**

```yaml
all:
  hosts:
    server:
      ansible_host: 192.168.1.10       # ← ваш IP
      ansible_user: ubuntu
```

## Шаг 3. Переменные и секреты

**group_vars/all/vars.yml:**

```yaml
---
# Пользователи
admin_user: deploy
admin_groups: sudo,docker
admin_shell: /bin/bash
ssh_public_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"

# SSH
ssh_port: 22
ssh_permit_root: "no"
ssh_password_auth: "no"

# Firewall
ufw_allowed_ports:
  - { port: "{{ ssh_port }}", proto: tcp }
  - { port: 80, proto: tcp }
  - { port: 443, proto: tcp }

# Docker
docker_users: ["deploy"]
docker_log_max_size: "50m"

# Обновления
enable_auto_updates: true
```

**group_vars/all/vault.yml** — создаём зашифрованный файл:

```bash
ansible-vault create group_vars/all/vault.yml
```

Содержимое:

```yaml
---
vault_admin_password: "$6$rounds=4096$..."   # mkpasswd --method=sha-512
```

## Шаг 4. SSH-шаблон

**templates/sshd_config.j2:**

```
# Managed by Ansible — DO NOT EDIT
Port {{ ssh_port }}
PermitRootLogin {{ ssh_permit_root }}
PasswordAuthentication {{ ssh_password_auth }}
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

## Шаг 5. Главный Playbook

**site.yml:**

```yaml
---
- name: Production Server Setup
  hosts: all
  become: yes

  tasks:
    # ========== 1. СИСТЕМА ==========
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install base packages
      ansible.builtin.apt:
        name:
          - curl
          - wget
          - git
          - vim
          - htop
          - unzip
          - ufw
          - fail2ban
          - unattended-upgrades
        state: present

    - name: Set timezone
      community.general.timezone:
        name: UTC

    # ========== 2. ПОЛЬЗОВАТЕЛИ ==========
    - name: Create admin user
      ansible.builtin.user:
        name: "{{ admin_user }}"
        groups: "{{ admin_groups }}"
        shell: "{{ admin_shell }}"
        create_home: yes
        password: "{{ vault_admin_password }}"

    - name: Add SSH key for admin
      ansible.posix.authorized_key:
        user: "{{ admin_user }}"
        key: "{{ ssh_public_key }}"
        exclusive: yes                     # удалить все остальные ключи

    - name: Allow admin passwordless sudo
      ansible.builtin.copy:
        content: "{{ admin_user }} ALL=(ALL) NOPASSWD:ALL"
        dest: "/etc/sudoers.d/{{ admin_user }}"
        mode: '0440'
        validate: "visudo -cf %s"

    # ========== 3. SSH HARDENING ==========
    - name: Deploy SSH config
      ansible.builtin.template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
        validate: "sshd -t -f %s"        # валидация перед применением!
        backup: yes
      notify: Restart SSH

    # ========== 4. FIREWALL (UFW) ==========
    - name: Allow required ports
      community.general.ufw:
        rule: allow
        port: "{{ item.port | string }}"
        proto: "{{ item.proto }}"
      loop: "{{ ufw_allowed_ports }}"

    - name: Enable UFW with deny policy
      community.general.ufw:
        state: enabled
        policy: deny
        logging: "on"

    # ========== 5. DOCKER ==========
    - name: Install Docker prerequisites
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

    - name: Install Docker Engine
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
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
              "max-file": "3"
            },
            "no-new-privileges": true,
            "live-restore": true,
            "userland-proxy": false
          }
        dest: /etc/docker/daemon.json
        mode: '0644'
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

    # ========== 6. АВТООБНОВЛЕНИЯ ==========
    - name: Enable unattended upgrades
      ansible.builtin.copy:
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        mode: '0644'
      when: enable_auto_updates

    # ========== 7. ПРОВЕРКА ==========
    - name: Verify Docker
      ansible.builtin.command: docker --version
      register: docker_ver
      changed_when: false

    - name: Verify UFW
      ansible.builtin.command: ufw status
      register: ufw_status
      changed_when: false

    - name: Summary
      ansible.builtin.debug:
        msg:
          - "✅ User: {{ admin_user }} created"
          - "✅ SSH: root login disabled, key-only auth"
          - "✅ UFW: enabled ({{ ufw_allowed_ports | length }} ports open)"
          - "✅ Docker: {{ docker_ver.stdout }}"

  handlers:
    - name: Restart SSH
      ansible.builtin.systemd:
        name: ssh
        state: restarted

    - name: Restart Docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
        daemon_reload: yes
```

## Шаг 6. Запуск

```bash
# Syntax check
ansible-playbook site.yml --syntax-check

# Dry run
ansible-playbook site.yml --check --diff --ask-vault-pass

# Полный запуск
ansible-playbook site.yml --ask-vault-pass

# Повторный запуск (идемпотентность)
ansible-playbook site.yml --ask-vault-pass
# Ожидание: changed=0
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| group_vars | Разделение `vars.yml` (открытые) и `vault.yml` (секреты) |
| vault | `ansible-vault create`, `--ask-vault-pass` |
| template | `sshd_config.j2` с переменными + `validate` |
| loops | `loop` для портов UFW и Docker-пользователей |
| when | Условное включение auto-updates |
| handlers | Restart SSH и Docker только при изменении конфигов |
| set_fact | Динамическое определение архитектуры |
| retries | `get_url` с 3 попытками для GPG-ключа |

## Что дальше

→ [[ansible/tutorials/03-roles-project]] — рефакторинг этого playbook в роли

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Сломали SSH-конфиг | Потеряли доступ к серверу | Всегда `validate: "sshd -t -f %s"` перед применением |
| Забыли `--ask-vault-pass` | `ERROR! Attempting to decrypt but no vault secrets found` | Добавить флаг или `vault_password_file` в ansible.cfg |
| UFW заблокировал SSH | Потеряли доступ | Правило для SSH **до** `ufw enable`. Используйте console/IPMI как fallback |
| Docker не стартует | `daemon.json` с ошибкой JSON | Валидировать JSON: `python3 -m json.tool < daemon.json` |