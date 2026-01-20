---
title: "Проект: Установка Docker через Ansible"
description: "Полный playbook для установки Docker Engine на Ubuntu 20.04+ с настройкой безопасности и пользователей."
---


Этот проект объединяет все изученные концепции (Tasks, Vars, Handlers, Error Handling) в один рабочий сценарий.

## Сценарий выполнения
1. Проверка совместимости ОС (только Ubuntu 20.04+).
2. Удаление старых версий Docker.
3. Установка репозитория и ключей GPG.
4. Установка Docker Engine, CLI и Compose.
5. Настройка `daemon.json` (логирование, драйверы).
6. Добавление текущего пользователя в группу `docker`.

## Полный Playbook (`docker_setup.yml`)

```yaml
***
- name: Setup Docker on Ubuntu
  hosts: all
  become: yes
  gather_facts: yes
  
  vars:
    docker_users: ["{{ ansible_user }}"]
    docker_log_max_size: "100m"
    docker_log_max_file: "10"
  
  pre_tasks:
    - name: Проверка совместимости ОС
      assert:
        that:
          - ansible_facts['distribution'] in ['Ubuntu', 'Debian']
          - ansible_facts['distribution_version'] is version('20.04', '>=')
        fail_msg: "Поддерживается только Ubuntu 20.04+ или Debian 11+"
  
  tasks:
    - name: Установка необходимых пакетов
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    - name: Загрузка GPG-ключа Docker
      get_url:
        url: "https://download.docker.com/linux/{{ ansible_facts['distribution'] | lower }}/gpg"
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'
      register: gpg_key

    - name: Добавление репозитория Docker
      apt_repository:
        repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/{{ ansible_facts['distribution'] | lower }} {{ ansible_facts['distribution_release'] }} stable"
        state: present
        filename: docker

    - name: Установка Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: yes
      notify: restart docker

    - name: Настройка daemon.json
      copy:
        content: |
          {
            "log-driver": "json-file",
            "log-opts": {
              "max-size": "{{ docker_log_max_size }}",
              "max-file": "{{ docker_log_max_file }}"
            }
          }
        dest: /etc/docker/daemon.json
        mode: '0600'
      notify: restart docker

    - name: Добавление пользователя в группу docker
      user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"

  handlers:
    - name: restart docker
      service:
        name: docker
        state: restarted
```

## Как запустить

```bash
# 1. Проверка синтаксиса
ansible-playbook -i hosts.ini docker_setup.yml --syntax-check

# 2. Тестовый прогон (Dry Run)
ansible-playbook -i hosts.ini docker_setup.yml --check

# 3. Боевой запуск
ansible-playbook -i hosts.ini docker_setup.yml
```

## Связанные материалы
- [[devops/ansible/tutorials/ansible-best-practices|Лучшие практики Ansible]]
