---
title: "Структура проекта"
type: reference
tags: [ansible, project, structure, directory, layout, roles, inventory, group-vars]
sources:
  docs: "https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html"
related:
  - "[[ansible/how-to/create-roles]]"
  - "[[ansible/how-to/manage-inventory]]"
  - "[[ansible/reference/best-practices]]"
---

# Структура Ansible-проекта

> **Справочник:** Рекомендуемая иерархия директорий для production-проектов.
> Масштабируется от одного playbook до многокомандного enterprise-проекта.

## Минимальный проект

```
my-project/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
└── site.yml
```

## Стандартный проект

```
ansible-project/
├── ansible.cfg                  # конфигурация (inventory path, forks, pipelining)
├── requirements.yml             # зависимости Galaxy (роли + коллекции)
├── Makefile                     # shortcuts: make deploy, make lint
│
├── inventory/
│   ├── production/
│   │   ├── hosts.yml            # хосты production
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml     # открытые переменные
│   │   │   │   └── vault.yml    # зашифрованные секреты
│   │   │   ├── webservers.yml
│   │   │   └── db.yml
│   │   └── host_vars/
│   │       └── web-01.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│           └── all/
│               ├── vars.yml
│               └── vault.yml
│
├── roles/
│   ├── common/                  # базовая настройка (users, ssh, ntp, firewall)
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── defaults/main.yml
│   │   └── templates/
│   ├── nginx/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── configure.yml
│   │   ├── handlers/main.yml
│   │   ├── defaults/main.yml
│   │   ├── templates/nginx.conf.j2
│   │   └── meta/main.yml       # зависимости
│   └── app/
│       ├── tasks/main.yml
│       ├── defaults/main.yml
│       └── templates/
│
├── playbooks/
│   ├── site.yml                 # главная точка входа (импортирует остальные)
│   ├── webservers.yml           # настройка веб-серверов
│   ├── db.yml                   # настройка БД
│   ├── deploy.yml               # деплой приложения
│   └── maintenance.yml          # обслуживание (backup, cleanup)
│
├── files/                       # глобальные статические файлы
│   └── ssh_keys/
│
├── templates/                   # глобальные шаблоны (вне ролей)
│
├── filter_plugins/              # кастомные Jinja2-фильтры
├── library/                     # кастомные модули
│
└── .gitignore
```

## site.yml — точка входа

```yaml
---
# site.yml — запускает все playbooks
- import_playbook: playbooks/webservers.yml
- import_playbook: playbooks/db.yml
```

```yaml
# playbooks/webservers.yml
- hosts: webservers
  become: yes
  roles:
    - common
    - nginx
    - app
```

## .gitignore

```gitignore
# Ansible
*.retry
.vault_pass
*.pyc
__pycache__/

# Collections (устанавливаются через requirements.yml)
collections/

# Чувствительные данные
*.pem
*.key
.env
```

## Makefile (shortcuts)

```makefile
.PHONY: lint deploy check

lint:
	ansible-lint playbooks/ roles/
	ansible-playbook playbooks/site.yml --syntax-check

check:
	ansible-playbook -i inventory/staging playbooks/site.yml --check --diff

deploy-staging:
	ansible-playbook -i inventory/staging playbooks/deploy.yml

deploy-production:
	ansible-playbook -i inventory/production playbooks/deploy.yml --ask-vault-pass

setup:
	ansible-galaxy install -r requirements.yml
	ansible-galaxy collection install -r requirements.yml
```

## Правила именования

| Элемент | Конвенция | Пример |
|---------|-----------|--------|
| Роли | snake_case, существительное | `nginx`, `postgres_server` |
| Переменные | `<role>_<param>` | `nginx_port`, `app_version` |
| Vault-переменные | `vault_<name>` | `vault_db_password` |
| Задачи | Глагол + объект | `Install Nginx`, `Deploy application` |
| Handlers | `Restart <service>` | `Restart Nginx`, `Reload PostgreSQL` |
| Теги | lowercase, kebab | `install`, `config`, `deploy` |
| Файлы inventory | `<environment>.yml` | `production.yml`, `staging.yml` |
