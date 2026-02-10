---
title: "Рецепт: Bootstrap сервера"
type: how-to
tags: [ansible, recipe, bootstrap, ubuntu, users, ssh, ufw, ntp, swap]
related:
  - "[[ansible/tutorials/02-server-setup]]"
  - "[[ansible/how-to/recipes/docker-install]]"
---

# Рецепт: Bootstrap Ubuntu-сервера

> Первичная настройка свежего сервера: пользователь, SSH-ключи, swap, timezone, базовые пакеты, firewall.

## Playbook

```yaml
---
- name: Bootstrap Fresh Server
  hosts: all
  become: yes
  gather_facts: yes

  vars:
    bootstrap_user: deploy
    bootstrap_ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    bootstrap_packages:
      - curl
      - wget
      - git
      - vim
      - htop
      - tmux
      - unzip
      - jq
      - ufw
      - fail2ban
    bootstrap_timezone: UTC
    bootstrap_swap_size: "2G"
    bootstrap_ssh_port: 22
    bootstrap_ufw_ports:
      - "{{ bootstrap_ssh_port }}/tcp"
      - "80/tcp"
      - "443/tcp"

  tasks:
    # === СИСТЕМА ===
    - name: Update all packages
      ansible.builtin.apt:
        upgrade: dist
        update_cache: yes
        cache_valid_time: 3600

    - name: Install base packages
      ansible.builtin.apt:
        name: "{{ bootstrap_packages }}"
        state: present

    - name: Set timezone
      community.general.timezone:
        name: "{{ bootstrap_timezone }}"

    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    # === SWAP ===
    - name: Check if swap exists
      ansible.builtin.command: swapon --show
      register: swap_check
      changed_when: false

    - name: Create swap file
      ansible.builtin.command: |
        fallocate -l {{ bootstrap_swap_size }} /swapfile
      when: swap_check.stdout == ""

    - name: Configure swap
      ansible.builtin.shell: |
        chmod 600 /swapfile
        mkswap /swapfile
        swapon /swapfile
        echo '/swapfile none swap sw 0 0' >> /etc/fstab
      when: swap_check.stdout == ""

    - name: Set swappiness
      ansible.posix.sysctl:
        name: vm.swappiness
        value: "10"
        sysctl_set: yes

    # === ПОЛЬЗОВАТЕЛЬ ===
    - name: Create admin user
      ansible.builtin.user:
        name: "{{ bootstrap_user }}"
        groups: sudo
        shell: /bin/bash
        create_home: yes

    - name: Add SSH key
      ansible.posix.authorized_key:
        user: "{{ bootstrap_user }}"
        key: "{{ bootstrap_ssh_key }}"

    - name: Passwordless sudo
      ansible.builtin.copy:
        content: "{{ bootstrap_user }} ALL=(ALL) NOPASSWD:ALL"
        dest: "/etc/sudoers.d/{{ bootstrap_user }}"
        mode: '0440'
        validate: "visudo -cf %s"

    # === SSH HARDENING ===
    - name: Harden SSH
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        validate: "sshd -t -f %s"
      loop:
        - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin no' }
        - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
        - { regexp: '^#?X11Forwarding', line: 'X11Forwarding no' }
        - { regexp: '^#?MaxAuthTries', line: 'MaxAuthTries 3' }
      notify: Restart SSH

    # === FIREWALL ===
    - name: Allow required ports
      community.general.ufw:
        rule: allow
        port: "{{ item.split('/')[0] }}"
        proto: "{{ item.split('/')[1] }}"
      loop: "{{ bootstrap_ufw_ports }}"

    - name: Enable UFW
      community.general.ufw:
        state: enabled
        policy: deny

    # === ПРОВЕРКА ===
    - name: Summary
      ansible.builtin.debug:
        msg:
          - "✅ User: {{ bootstrap_user }}"
          - "✅ SSH: root disabled, key-only"
          - "✅ UFW: {{ bootstrap_ufw_ports | length }} ports open"
          - "✅ Swap: {{ bootstrap_swap_size }}"
          - "✅ Timezone: {{ bootstrap_timezone }}"

  handlers:
    - name: Restart SSH
      ansible.builtin.systemd:
        name: ssh
        state: restarted
```

## Запуск

```bash
# Первый запуск (может потребоваться root-пароль)
ansible-playbook bootstrap.yml -u root

# Последующие запуски (через deploy-пользователя)
ansible-playbook bootstrap.yml -u deploy
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| UFW заблокировал SSH | Правило SSH **до** `ufw enable`. Используйте console для восстановления |
| SSH config сломан | `validate: "sshd -t -f %s"` предотвращает применение невалидного конфига |
| swap уже есть | `swapon --show` + `when` — идемпотентно |