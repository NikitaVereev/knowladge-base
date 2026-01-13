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
  
  vars:
    docker_compose_version: "2.24.0"
  
  tasks:
    # Подготовка
    - name: Update package cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Install required packages
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
    
    # Добавить Docker репозиторий
    - name: Create keyrings directory
      file:
        path: /etc/apt/keyrings
        state: directory
        mode: '0755'
    
    - name: Add Docker GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
        gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      args:
        creates: /etc/apt/keyrings/docker.gpg
    
    - name: Add Docker repository
      shell: |
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
        https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
        tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list
    
    - name: Update cache after adding Docker repo
      apt:
        update_cache: yes
    
    # Установка Docker
    - name: Install Docker Engine
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
    
    # Запуск сервисов
    - name: Start Docker service
      systemd:
        name: docker
        state: started
        enabled: yes
        daemon_reload: yes
    
    - name: Start containerd service
      systemd:
        name: containerd
        state: started
        enabled: yes
    
    # Настройка пользователя
    - name: Add current user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
    
    # Проверка установки
    - name: Check Docker version
      shell: "docker --version"
      register: docker_version
      changed_when: False
    
    - name: Display Docker version
      debug:
        msg: "{{ docker_version.stdout }}"
    
    - name: Test Docker installation
      shell: "docker run --rm hello-world"
      register: docker_test
      changed_when: False
    
    - name: Display test result
      debug:
        msg: "{{ docker_test.stdout_lines[-1] }}"
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
