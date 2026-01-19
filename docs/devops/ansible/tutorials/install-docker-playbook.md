---
title: "2 Playbook для установки Docker"
description: "Пример Ansible Playbook для автоматической установки Docker Engine, добавления репозиториев и настройки прав пользователя."
---


Вместо ручного ввода команд установки на каждом сервере, мы опишем состояние "Docker установлен" в виде Playbook.

## Плейбук `install-docker.yml`

Создайте файл `playbooks/install-docker.yml`:

```yaml
***
- name: Install Docker Engine
  hosts: all
  gather_facts: yes
  become: yes  # Выполнять задачи от root (sudo)
  
  tasks:
    - name: Remove old Docker packages
      apt:
        name:
          - docker.io
          - docker-compose
          - podman-docker
        state: absent

    - name: Install dependencies
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
        state: present
        update_cache: yes

    - name: Create keyrings directory
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'

    - name: Add Docker GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /etc/apt/keyrings/docker.asc
      args:
        creates: /etc/apt/keyrings/docker.asc  # Не выполнять, если файл уже есть

    - name: Add Docker repository
      shell: |
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
        https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Install Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: Start and enable Docker service
      systemd:
        name: docker
        enabled: yes
        state: started

    - name: Add current user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
```

## Запуск

```bash
ansible-playbook playbooks/install-docker.yml
```

## Проверка

После выполнения плейбука проверьте версию Docker на всех хостах одной командой (ad-hoc):

```bash
ansible all -m command -a "docker --version"
```

## Связанные материалы

- [[devops/docker/how-to/install-ubuntu-2404|Ручная установка Docker]]
- [[devops/ansible/explanation/modules|Как работают модули Ansible]]
