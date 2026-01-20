---
title: "Настройка файла инвентаря"
description: "Примеры конфигурации hosts.yaml, параметры подключения и отладка."
---


Пример настройки статического инвентаря в формате YAML с различными параметрами подключения.

## Пример структуры (`hosts.yaml`)

```yaml
all:
  # Общие переменные для всех хостов
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    ansible_python_interpreter: /usr/bin/python3
  
  children:
    # Группа веб-серверов
    webservers:
      hosts:
        # Хост с явным указанием IP (если DNS не настроен)
        web-01:
          ansible_host: 192.168.1.10
        
        # Хост с нестандартным SSH портом
        web-02:
          ansible_host: 192.168.1.11
          ansible_port: 2222

    # Группа баз данных
    databases:
      hosts:
        db-primary:
          ansible_host: 10.0.0.5

    # Локальное подключение (без SSH)
    local:
      hosts:
        localhost:
          ansible_connection: local
```

## Параметры подключения

| Переменная | Описание |
|------------|----------|
| `ansible_host` | Реальный IP или домен сервера. Если не указан, используется имя хоста из инвентаря. |
| `ansible_port` | Порт SSH (по умолчанию 22). |
| `ansible_user` | Пользователь, под которым Ansible будет заходить на сервер. |
| `ansible_ssh_private_key_file` | Путь к приватному ключу SSH. |
| `ansible_connection` | Тип соединения (`ssh`, `local`, `docker`). |
| `ansible_become` | `true`/`yes` — выполнять команды через sudo. |

## Проверка инвентаря

1. **Показать структуру в виде графа:**
   ```bash
   ansible-inventory -i hosts.yaml --graph
   ```

2. **Проверить доступность хостов (ping):**
   ```bash
   ansible -i hosts.yaml all -m ping
   ```

3. **Посмотреть переменные конкретного хоста:**
   ```bash
   ansible-inventory -i hosts.yaml --host web-01
   ```

## Связанные материалы
- [[devops/ansible/explanation/inventory-basics|Основы инвентаря]]
- [[devops/ansible/how-to/install-and-setup|Установка Ansible]]
