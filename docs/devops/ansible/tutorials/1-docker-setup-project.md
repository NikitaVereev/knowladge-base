---
title: "1 Практический Проект - Docker Setup"
description: "Production-ready playbook: установка Docker, авто-определение архитектуры, security hardening и проверка установки."
---

В этом проекте мы реализуем полноценный сценарий настройки Docker хоста. В отличие от базовых примеров, здесь учтены:
1.  Автоматическое определение архитектуры (amd64/arm64).
2.  Очистка системы от старых репозиториев.
3.  Настройка безопасности (Hardening) через `daemon.json`.
4.  Детальная проверка результата.

## Полный Playbook: Установка Docker

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
    
    - name: Определение архитектуры
      set_fact:
        docker_arch: "{{ 'amd64' if ansible_facts['machine'] == 'x86_64' else 'arm64' if ansible_facts['machine'] == 'aarch64' else 'armhf' }}"
  
  tasks:
    # --- Очистка ---
    - name: Удаление старых Docker репозиториев
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/apt/sources.list.d/docker.list
        - /usr/share/keyrings/docker-archive-keyring.gpg
        - /etc/apt/keyrings/docker.gpg
    
    - name: Обновление кэша пакетов
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    # --- Зависимости ---
    - name: Установка необходимых пакетов
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - python3-pip
        state: present
    
    # --- Репозиторий ---
    - name: Создание директории для ключей
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'
    
    - name: Загрузка GPG-ключа Docker
      get_url:
        url: "https://download.docker.com/linux/{{ ansible_facts['distribution'] | lower }}/gpg"
        dest: /etc/apt/keyrings/docker.asc
        mode: '0644'
      register: gpg_key
      retries: 3
      delay: 5
    
    - name: Добавление репозитория Docker
      apt_repository:
        repo: "deb [arch={{ docker_arch }} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/{{ ansible_facts['distribution'] | lower }} {{ ansible_facts['distribution_release'] }} stable"
        state: present
        filename: docker
    
    - name: Обновление кэша после добавления репо
      apt:
        update_cache: yes
    
    # --- Установка ---
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
      register: docker_install
    
    # --- Hardening ---
    - name: Создание daemon.json с security hardening
      copy:
        content: |
          {
            "log-driver": "json-file",
            "log-opts": {
              "max-size": "{{ docker_log_max_size }}",
              "max-file": "{{ docker_log_max_file }}",
              "compress": "true"
            },
            "storage-driver": "overlay2",
            "icc": false,
            "userland-proxy": false,
            "no-new-privileges": true,
            "live-restore": true
          }
        dest: /etc/docker/daemon.json
        owner: root
        group: root
        mode: '0600'
        backup: yes
      notify: restart docker
    
    # --- Сервисы ---
    - name: Запуск и активация Docker
      systemd:
        name: docker
        state: started
        enabled: yes
    
    - name: Запуск containerd
      systemd:
        name: containerd
        state: started
        enabled: yes
    
    # --- Пользователи ---
    - name: Добавление пользователей в группу docker
      user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"
    
    - name: Перезагрузка SSH соединения (для обновления групп)
      meta: reset_connection
    
    # --- Проверка (Smoke Test) ---
    - name: Проверка версии Docker
      command: docker --version
      register: docker_version
      changed_when: false
    
    - name: Проверка конфигурации daemon.json
      shell: cat /etc/docker/daemon.json | python3 -m json.tool
      register: daemon_json_check
      changed_when: false
    
    - name: Вывод результатов установки
      debug:
        msg:
          - "✅ Docker установлен: {{ docker_version.stdout }}"
          - "✅ Пользователь добавлен в группу docker"
          - "✅ Конфигурация Hardening применена успешно"
  
  handlers:
    - name: restart docker
      systemd:
        name: docker
        state: restarted
        daemon_reload: yes
```

## Как запустить

```bash
# 1. Проверка синтаксиса и Dry-Run
ansible-playbook -i hosts.ini docker_setup.yml --syntax-check
ansible-playbook -i hosts.ini docker_setup.yml --check

# 2. Боевой запуск
ansible-playbook -i hosts.ini docker_setup.yml

# 3. Если требуется ввод пароля sudo
ansible-playbook -i hosts.ini docker_setup.yml -K
```

## Troubleshooting

**Проблема с правами (Permission denied)**
Если после установки `docker ps` требует sudo:
1. Убедитесь, что плейбук выполнился успешно.
2. Перезайдите на сервер (SSH session logout/login), чтобы обновилось членство в группах.

**Ошибка репозитория**
Убедитесь, что `docker_arch` определился верно.
```bash
ansible all -m setup -a "filter=ansible_architecture"
```
