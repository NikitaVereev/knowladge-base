---
title: "Управление инвентарём"
type: how-to
tags: [ansible, inventory, hosts, ssh, groups, dynamic-inventory, yaml, ini]
sources:
  docs: "https://docs.ansible.com/ansible/latest/inventory_guide/index.html"
related:
  - "[[ansible/explanation/inventory]]"
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/how-to/install-and-configure]]"
---

# Управление инвентарём

> **TL;DR:** Inventory в YAML → `group_vars/` для переменных → SSH-ключи через `ssh-copy-id`.
> `ansible-inventory --graph` — визуализация. `ansible all -m ping` — проверка.

## Создание inventory

### Структура файлов

```
inventory/
├── hosts.yml                    # основной файл инвентаря
├── group_vars/
│   ├── all.yml                  # переменные для ВСЕХ хостов
│   ├── webservers.yml           # переменные для группы
│   └── db.yml
└── host_vars/
    └── web-01.yml               # переменные для конкретного хоста
```

### Пример hosts.yml

```yaml
all:
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    ansible_python_interpreter: /usr/bin/python3

  children:
    webservers:
      hosts:
        web-01:
          ansible_host: 192.168.1.10
        web-02:
          ansible_host: 192.168.1.11
          ansible_port: 2222

    databases:
      hosts:
        db-primary:
          ansible_host: 10.0.0.5

    local:
      hosts:
        localhost:
          ansible_connection: local

    # Группа групп
    production:
      children:
        webservers:
        databases:
```

### Переменные группы (group_vars/webservers.yml)

```yaml
---
http_port: 80
app_version: "2.1.0"
nginx_worker_processes: auto
```

## Настройка SSH

### 1. Генерация ключа (на Control Node)

```bash
ssh-keygen -t ed25519 -C "ansible-control"
# Enter для всех вопросов (без passphrase для автоматизации)
```

### 2. Копирование ключа на серверы

```bash
# Один сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10

# Все серверы из inventory (быстрый скрипт)
for host in 192.168.1.10 192.168.1.11 10.0.0.5; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@$host
done
```

### 3. Проверка (пароль не должен запрашиваться)

```bash
ssh ubuntu@192.168.1.10
```

### SSH Troubleshooting

| Проблема | Решение |
|----------|---------|
| `Permission denied (publickey)` | Проверить: ключ добавлен в `~/.ssh/authorized_keys`? Права `700` на `.ssh/`, `600` на `authorized_keys` |
| `Host key verification failed` | `host_key_checking = False` в ansible.cfg или `ssh-keyscan host >> known_hosts` |
| Нужен пароль sudo | `ansible-playbook site.yml -K` или `ansible_become_pass` в vault |

## Параметры подключения

| Переменная | Описание | По умолчанию |
|------------|----------|-------------|
| `ansible_host` | IP или домен | имя хоста |
| `ansible_port` | SSH-порт | 22 |
| `ansible_user` | SSH-пользователь | текущий |
| `ansible_ssh_private_key_file` | Путь к ключу | ~/.ssh/id_rsa |
| `ansible_connection` | Тип: `ssh`, `local`, `docker`, `winrm` | ssh |
| `ansible_become` | Использовать sudo | false |
| `ansible_become_method` | Метод: `sudo`, `su`, `doas` | sudo |
| `ansible_python_interpreter` | Путь к Python | auto |

## Проверка инвентаря

```bash
# Граф групп и хостов
ansible-inventory -i inventory/hosts.yml --graph

# Все переменные хоста (с учётом group_vars + host_vars)
ansible-inventory -i inventory/hosts.yml --host web-01

# Ping всех хостов
ansible all -m ping

# Ping конкретной группы
ansible webservers -m ping

# Список хостов (без подключения)
ansible all --list-hosts
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Переменные внутри hosts.yml | Нечитаемый файл при 20+ хостах | Вынести в `group_vars/` и `host_vars/` |
| Один inventory для dev и prod | Случайный деплой на prod | Разделить: `inventory/dev/hosts.yml`, `inventory/prod/hosts.yml` |
| SSH-ключ с passphrase | `ansible all -m ping` зависает | Использовать `ssh-agent`: `eval $(ssh-agent) && ssh-add` |
| `ansible_python_interpreter` не указан | `MODULE FAILURE: /usr/bin/python not found` | Указать `/usr/bin/python3` в `group_vars/all.yml` |