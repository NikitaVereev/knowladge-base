---
title: 01 Roles & Organization
---

---

## Roles Структура

```
my-role/
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   └── configure.yml
├── handlers/
│   └── main.yml
├── vars/
│   └── main.yml
├── defaults/
│   └── main.yml
├── files/
├── templates/
├── tests/
│   └── inventory
└── README.md
```

---

## Collections

**Использование collections:**
```bash
# Установить из Galaxy
ansible-galaxy collection install community.general

# Или через requirements.yml
---
collections:
  - community.general
  - community.docker
  - kubernetes.core
```

**В playbook:**
```yaml
---
- hosts: all
  collections:
    - community.general
    - community.docker

  tasks:
    - name: Manage systemd service
      systemd:
        name: nginx
        state: started

    - name: Manage Docker container
      docker_container:
        name: web
        image: nginx
        state: started
```

---

## Создание Collection

```bash
# Инициализировать
ansible-galaxy collection init my_company.infrastructure

# Структура
collections/
├── my_company/
│   └── infrastructure/
│       ├── roles/
│       │   ├── nginx/
│       │   └── postgresql/
│       ├── plugins/
│       │   ├── filter_plugins/
│       │   ├── action_plugins/
│       │   └── lookup_plugins/
│       ├── README.md
│       └── galaxy.yml
```

**galaxy.yml:**
```yaml
---
namespace: my_company
name: infrastructure
version: 1.0.0
description: Infrastructure automation
authors:
  - DevOps Team
```

---

## Molecule для Testing

```bash
# Инициализировать
molecule init role my-nginx

# Запустить тесты
molecule test

# Команды
molecule converge   # Применить role
molecule verify     # Проверить результат
molecule destroy    # Очистить окружение
```

**molecule.yml:**
```yaml
---
driver:
  name: docker

platforms:
  - name: ubuntu-22.04
    image: ubuntu:22.04
    pre_build_image: true

  - name: debian-11
    image: debian:11

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml
    verify: verify.yml
```

**verify.yml:**
```yaml
---
- name: Verify role
  hosts: all
  gather_facts: false

  tasks:
    - name: Check nginx is installed
      command: nginx -v
      changed_when: false

    - name: Check nginx is running
      systemd:
        name: nginx
        state: started
      register: nginx_service
      failed_when: nginx_service is not defined
```

---

## Role Dependencies

**meta/main.yml:**
```yaml
---
galaxy_info:
  namespace: my_company
  name: infrastructure
  version: 1.0.0
  description: Infrastructure automation
  license: MIT

dependencies:
  - role: community.general.timezone
  - role: geerlingguy.firewall
    vars:
      firewall_allowed_tcp_ports: [22, 80, 443]
```

---

## Best Practices

**Организация:**
- Один role = одна ответственность
- Параметризуемые через variables
- Переиспользуемые в разных проектах
- Хорошая документация

**Тестирование:**
- Molecule для каждого role
- Проверка на разных ОС
- Integration tests
- CI/CD pipeline

---

**Следующее:** [[02-execution-advanced|Advanced Execution Control]]
