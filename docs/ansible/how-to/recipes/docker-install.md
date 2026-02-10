---
title: "Рецепт: Установка Docker"
type: how-to
tags: [ansible, recipe, docker, install, daemon-json, compose, buildkit]
related:
  - "[[ansible/how-to/recipes/server-bootstrap]]"
  - "[[ansible/how-to/recipes/deploy-docker-app]]"
  - "[[docker/how-to/install]]"
---

# Рецепт: Установка Docker Engine

> Production-ready установка Docker CE на Ubuntu/Debian: GPG-ключ, репозиторий, daemon.json hardening, Compose plugin.

## Playbook

```yaml
---
- name: Install Docker Engine
  hosts: all
  become: yes

  vars:
    docker_users: ["deploy"]
    docker_log_max_size: "50m"
    docker_log_max_file: "3"
    docker_default_address_pools:
      - base: "172.17.0.0/12"
        size: 24

  tasks:
    - name: Remove old Docker packages
      ansible.builtin.apt:
        name:
          - docker
          - docker-engine
          - docker.io
          - containerd
          - runc
        state: absent

    - name: Install prerequisites
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    - name: Create keyrings directory
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker GPG key
      ansible.builtin.get_url:
        url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"
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
          https://download.docker.com/linux/{{ ansible_distribution | lower }}
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
              "max-file": "{{ docker_log_max_file }}"
            },
            "default-address-pools": {{ docker_default_address_pools | to_json }},
            "no-new-privileges": true,
            "live-restore": true,
            "userland-proxy": false
          }
        dest: /etc/docker/daemon.json
        mode: '0644'
      notify: Restart Docker

    - name: Start and enable Docker
      ansible.builtin.systemd:
        name: docker
        state: started
        enabled: yes

    - name: Add users to docker group
      ansible.builtin.user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"

    - name: Verify installation
      ansible.builtin.command: docker info --format '{{ '{{' }}.ServerVersion{{ '}}' }}'
      register: docker_version
      changed_when: false

    - name: Show result
      ansible.builtin.debug:
        msg: "✅ Docker {{ docker_version.stdout }} installed"

  handlers:
    - name: Restart Docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
        daemon_reload: yes
```

## Роль (переиспользуемая)

Для использования в нескольких проектах, вынесите в `roles/docker/` с `defaults/main.yml`:

```yaml
# roles/docker/defaults/main.yml
docker_users: []
docker_log_max_size: "50m"
docker_log_max_file: "3"
docker_default_address_pools:
  - base: "172.17.0.0/12"
    size: 24
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Старый Docker из snap/apt | Task «Remove old packages» удалит устаревшие версии |
| daemon.json с невалидным JSON | Ansible упадёт на `copy`. Протестируйте содержимое `content` |
| `docker_users` пусто | Нет ошибки — loop не выполнится. Docker доступен только через sudo |
| Конфликт подсетей с VPN | Настроить `default-address-pools` через переменную |
