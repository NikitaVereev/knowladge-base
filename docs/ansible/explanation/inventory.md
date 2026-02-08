---
title: "Ansible Inventory"
type: explanation
tags: [ansible, inventory, hosts, groups, group-vars, host-vars, dynamic-inventory, yaml, ini]
sources:
  docs: "https://docs.ansible.com/ansible/latest/inventory_guide/index.html"
related:
  - "[[ansible/explanation/architecture]]"
  - "[[ansible/explanation/variables-and-facts]]"
  - "[[ansible/how-to/manage-inventory]]"
  - "[[ansible/reference/project-structure]]"
---

# Ansible Inventory

> **TL;DR:** Inventory — источник истины: какие серверы есть, как к ним подключаться,
> какие переменные использовать. YAML-формат предпочтителен. Сложные переменные — в `group_vars/` и `host_vars/`.

Инвентарь (Inventory) описывает **какими** серверами нужно управлять, **как** к ним подключаться и какие **переменные** использовать.

## Форматы файлов

### INI (Классический)

Простой формат для быстрых тестов. Плохо читается при сложной вложенности.

```ini
[webservers]
web1.example.com
web2.example.com ansible_port=2222

[db]
db1.example.com

[production:children]
webservers
db
```

### YAML (Рекомендуемый)

Структурированный формат с поддержкой сложных типов данных (списки, словари).

```yaml
all:
  vars:
    ansible_user: admin             # общая переменная для всех
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
          ansible_port: 2222        # переопределение для конкретного хоста
      vars:
        http_port: 80
    db:
      hosts:
        db1.example.com:
    production:
      children:                     # группа групп
        webservers:
        db:
```

## Группы и переменные

### Иерархия групп

```
all                    ← неявная корневая группа (все хосты)
├── ungrouped          ← хосты без явной группы
├── webservers
│   ├── web1
│   └── web2
├── db
│   └── db1
└── production         ← parent group (children: webservers + db)
    ├── webservers
    └── db
```

Хост может входить в несколько групп одновременно. Группы могут вкладываться (`children`).

### Где определять переменные

| Место | Приоритет | Когда использовать |
|-------|-----------|-------------------|
| Внутри inventory файла (`vars:`) | Низкий | Быстрые тесты |
| `group_vars/all.yaml` | Низкий | Общие настройки (NTP, DNS, SSH user) |
| `group_vars/<group>.yaml` | Средний | Настройки группы (порт БД, версия приложения) |
| `host_vars/<host>.yaml` | Высокий | Уникальные настройки хоста (IP, сертификаты) |

Файловая структура:

```
inventory/
├── hosts.yaml
├── group_vars/
│   ├── all.yaml            # переменные для ВСЕХ хостов
│   ├── webservers.yaml     # переменные для группы webservers
│   └── db.yaml
└── host_vars/
    ├── web1.example.com.yaml
    └── db1.example.com.yaml
```

## Динамический инвентарь

Для облачных сред (AWS, GCP, Azure) хосты создаются и уничтожаются динамически. Статический файл не подходит.

```yaml
# inventory/aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions:
  - eu-central-1
keyed_groups:
  - key: tags.Environment        # группировать по тегу Environment
    prefix: env
  - key: instance_type
    prefix: type
filters:
  tag:Project: myapp
```

```bash
# Проверить что видит динамический инвентарь
ansible-inventory -i inventory/aws_ec2.yaml --graph
```

## Специальные переменные подключения

```yaml
# Переменные, влияющие на подключение к хосту
ansible_host: 10.0.1.5           # IP (если отличается от имени)
ansible_port: 2222               # SSH-порт
ansible_user: deploy             # SSH-пользователь
ansible_ssh_private_key_file: ~/.ssh/deploy_key
ansible_python_interpreter: /usr/bin/python3
ansible_become: true             # использовать sudo
ansible_become_method: sudo
ansible_become_pass: "{{ vault_sudo_password }}"
```

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «Переменные в inventory — удобно» | При росте проекта становится нечитаемо. Используй `group_vars/` и `host_vars/` |
| «Группа `all` не нужна» | `all` — неявная группа, в неё входят все хосты. `group_vars/all.yaml` — лучшее место для общих настроек |
| «Один inventory на все окружения» | Разделяй: `inventory/dev/`, `inventory/prod/`. Или используй переменные группы для разграничения |
| «Динамический инвентарь сложный» | С плагином `aws_ec2` / `gcp_compute` — 10 строк YAML. Всегда актуальный список хостов |
