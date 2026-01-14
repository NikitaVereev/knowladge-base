---
title: 05 Практический Проект - Docker Setup
---

---

## Полный Playbook: Установка Docker

```yaml
---
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
    # Очистка старых конфигураций (если есть)
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
    
    # Настройка репозитория Docker
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
    
    # Установка Docker
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
    
    # Настройка безопасности (без ulimits в daemon.json)
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
    
    # Управление сервисами
    - name: Перезагрузка systemd демона
      systemd:
        daemon_reload: yes
    
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
    
    # Настройка пользователей
    - name: Добавление пользователей в группу docker
      user:
        name: "{{ item }}"
        groups: docker
        append: yes
      loop: "{{ docker_users }}"
      register: user_added
    
    - name: Перезагрузка SSH соединения
      meta: reset_connection
    
    # Проверка установки
    - name: Проверка версии Docker
      command: docker --version
      register: docker_version
      changed_when: false
    
    - name: Проверка Docker Compose
      command: docker compose version
      register: compose_version
      changed_when: false
    
    - name: Получение информации о Docker
      shell: docker info --format json
      register: docker_info_raw
      changed_when: false
    
    - name: Проверка конфигурации daemon.json
      shell: cat /etc/docker/daemon.json | python3 -m json.tool
      register: daemon_json_check
      changed_when: false
    
    - name: Вывод результатов установки
      debug:
        msg:
          - "✅ Docker установлен: {{ docker_version.stdout }}"
          - "✅ Compose установлен: {{ compose_version.stdout }}"
          - "✅ Пользователь добавлен в группу docker"
          - "✅ Конфигурация применена успешно"
          - ""
          - "Проверь установку командой:"
          - "  docker run --rm hello-world"
  
  handlers:
    - name: restart docker
      systemd:
        name: docker
        state: restarted
        daemon_reload: yes

```

---

## Запуск Playbook

```bash
# Dry-run перед выполнением
ansible-playbook -i hosts.ini docker_setup.yml --check

# Выполнить
ansible-playbook -i hosts.ini docker_setup.yml

# С подробным выводом
ansible-playbook -i hosts.ini docker_setup.yml -vvv

# На конкретной группе
ansible-playbook -i hosts.ini docker_setup.yml -l webservers

# С запросом пароля
ansible-playbook -i hosts.ini docker_setup.yml -K
```

---

## Проверка После Выполнения

```bash
# SSH на сервер
ssh ubuntu@server

# Проверить Docker
docker --version
docker ps
docker run hello-world

# Проверить пользователя в группе docker
groups
# Должно вывести: ubuntu docker ...

# Тест без sudo
docker ps
# Должно работать без sudo
```

---

## Обработка Ошибок (Расширенная версия)

```yaml
---
- name: Setup Docker with error handling
  hosts: all
  become: yes
  
  pre_tasks:
    - name: Check Ubuntu version
      assert:
        that:
          - ansible_distribution == "Ubuntu"
          - ansible_distribution_version is version('20.04', '>=')
        fail_msg: "Only Ubuntu 20.04+ is supported"
  
  tasks:
    - block:
        - name: Update packages
          apt:
            update_cache: yes
          async: 300
          poll: 10
        
        - name: Install Docker
          apt:
            name:
              - docker-ce
              - docker-compose-plugin
            state: present
        
        - name: Start Docker
          systemd:
            name: docker
            state: started
            enabled: yes
      
      rescue:
        - name: Log error
          debug:
            msg: "Docker setup failed: {{ ansible_failed_result }}"
        
        - name: Remove Docker if partially installed
          apt:
            name: docker-ce
            state: absent
        
        - fail:
            msg: "Docker installation failed"
      
      always:
        - name: Final status
          debug:
            msg: "Docker setup completed"
```

---

## Проверка Качества Кода

**ansible-lint:**
```bash
# Установить
pip install ansible-lint

# Проверить playbook
ansible-lint docker_setup.yml

# Проверить всё в директории
ansible-lint .
```

**.ansible-lint конфиг:**
```yaml
---
skip_list:
  - "line-too-long"
  - "trailing-whitespace"

exclude_paths:
  - roles/external/
```

---

## Best Practices

✓ **Используй `changed_when: False` для команд без изменений**
```yaml
- shell: "docker --version"
  register: version
  changed_when: False
```

✓ **Используй `register` для проверки результатов**
```yaml
- shell: "docker run hello-world"
  register: test_result
```

✓ **Используй `debug` для вывода информации**
```yaml
- debug:
    msg: "{{ version.stdout }}"
```

✓ **Используй `async` для долгих операций**
```yaml
- apt: upgrade=full
  async: 600
  poll: 10
```

✓ **Всегда используй `--check` перед запуском**
```bash
ansible-playbook playbook.yml --check
```

---

## CI/CD Integration

**GitHub Actions (example):**
```yaml
name: Deploy Docker

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Ansible playbook
        run: |
          pip install ansible
          ansible-playbook docker_setup.yml \
            -i inventory.ini \
            --key-file ~/.ssh/id_rsa
```

---

## Troubleshooting

**Docker permission denied:**
```bash
# Добавить пользователя в группу
sudo usermod -aG docker $USER
newgrp docker

# Или выполнить проверку через новый shell
ssh ubuntu@server
docker ps
```

**Repository error:**
```bash
# Убедиться что GPG ключ скопирован правильно
ls -la /etc/apt/keyrings/docker.gpg

# Проверить sources.list
cat /etc/apt/sources.list.d/docker.list
```

**Playbook failed:**
```bash
# Запустить с подробным выводом
ansible-playbook docker_setup.yml -vvv

# Проверить SSH подключение
ansible all -m ping

# Проверить Python
ansible all -m setup | grep python
```

---
