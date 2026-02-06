---
title: "2 Настройка файла инвентаря"
description: "Примеры конфигурации hosts.yaml, параметры подключения и отладка."
---

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
        # Хост с явным указанием IP
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

## Параметры подключения (Magic Variables)

| Переменная | Описание |
|------------|----------|
| `ansible_host` | Реальный IP или домен сервера. |
| `ansible_port` | Порт SSH (по умолчанию 22). |
| `ansible_user` | Пользователь SSH. |
| `ansible_ssh_private_key_file` | Путь к ключу. |
| `ansible_connection` | Тип соединения: `ssh` (default), `local`, `docker` (для управления контейнерами), `winrm` (для Windows). |
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
