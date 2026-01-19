---
title: 03 Инвентарь и Хосты
---

---

## Что такое Инвентарь

**Файл со списком целевых хостов:**
- IP адреса или хостнеймы
- Группы (webservers, databases, all)
- Параметры подключения к каждому хосту
- Переменные для хостов и групп

---

## INI Формат (Простой)

**hosts.ini:**
```ini
[webservers]
web1.example.com
web2.example.com
web3.example.com ansible_user=deploy

[databases]
db1.example.com ansible_port=3306
db2.example.com ansible_port=3306

[loadbalancers]
lb1.example.com

# Переменные для группы
[webservers:vars]
app_port=8080
app_env=production

# Переменные для всех хостов
[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_become=yes
ansible_become_method=sudo
```

---

## YAML Формат (Гибкий)

**hosts.yaml:**
```yaml
---
all:
  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    ansible_become: yes
    ansible_become_method: sudo
    ansible_python_interpreter: /usr/bin/python3
  
  children:
    webservers:
      vars:
        app_port: 8080
        app_env: production
      hosts:
        web1.example.com:
        web2.example.com:
        web3.example.com:
          ansible_user: deploy
    
    databases:
      vars:
        db_port: 3306
      hosts:
        db1.example.com:
        db2.example.com:
    
    loadbalancers:
      hosts:
        lb1.example.com:
```

---

## Параметры Подключения

| Параметр | Пример | Описание |
|----------|--------|---------|
| `ansible_host` | `192.168.1.10` | IP или хостнейм (если другой чем название) |
| `ansible_port` | `2222` | SSH порт (по умолчанию 22) |
| `ansible_user` | `deploy` | SSH пользователь (по умолчанию root или текущий) |
| `ansible_ssh_private_key_file` | `~/.ssh/id_ed25519` | Путь к приватному SSH ключу |
| `ansible_become` | `yes` | Использовать sudo/su для выполнения |
| `ansible_become_method` | `sudo` | Метод повышения прав (sudo, su, doas) |
| `ansible_become_user` | `root` | Пользователь для повышения прав |
| `ansible_python_interpreter` | `/usr/bin/python3.12` | Путь к Python на целевом хосте |

---

## Практический Пример

```yaml
---
all:
  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    ansible_become: yes
    ansible_become_method: sudo
  
  children:
    # Веб-серверы
    webservers:
      vars:
        http_port: 80
        https_port: 443
        nginx_workers: 4
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: ubuntu
        
        web2:
          ansible_host: 192.168.1.11
          ansible_user: ubuntu
        
        web3:
          ansible_host: 192.168.1.12
          ansible_user: ubuntu
          ansible_port: 2222  # нестандартный порт
    
    # Базы данных
    databases:
      vars:
        db_backup_retention: 7
      hosts:
        db_primary:
          ansible_host: 192.168.2.10
          ansible_user: devops
          ansible_port: 3306
        
        db_replica:
          ansible_host: 192.168.2.11
          ansible_user: devops
          ansible_port: 3306
    
    # Локальная машина (управления)
    control_plane:
      hosts:
        localhost:
          ansible_connection: local
```

---

## Использование Инвентаря

**Пинг все хосты:**
```bash
ansible -i hosts.yaml all -m ping
```

**Пинг конкретную группу:**
```bash
ansible -i hosts.yaml webservers -m ping
```

**Информация о конфигурации:**
```bash
# Список всех хостов
ansible-inventory -i hosts.yaml --list

# График групп
ansible-inventory -i hosts.yaml --graph

# Вывести переменные хоста
ansible-inventory -i hosts.yaml --host web1
```

**Выполнить команду:**
```bash
# На всех вебсерверах
ansible -i hosts.yaml webservers -m shell -a "uptime"

# На конкретном хосте
ansible -i hosts.yaml web1 -m setup | head -50
```

---

## Переменные в Инвентаре

**Для конкретного хоста:**
```yaml
webservers:
  hosts:
    web1:
      ansible_host: 192.168.1.10
      server_role: primary
      max_connections: 1000
```

**Для группы:**
```yaml
webservers:
  vars:
    app_env: production
    log_level: info
  hosts:
    web1: ...
    web2: ...
```

**Вложенные группы:**
```yaml
all:
  children:
    production:
      children:
        prod_web:
          hosts: ...
        prod_db:
          hosts: ...
    
    staging:
      children:
        stage_web:
          hosts: ...
```

---

## Тестирование Подключения

```bash
# Базовый пинг
ansible -i hosts.yaml all -m ping

# С подробным выводом
ansible -i hosts.yaml all -m ping -vvv

# Собрать факты (информацию о системе)
ansible -i hosts.yaml webservers -m setup | grep -E "ansible_distribution|ansible_processor_cores"

# Выполнить простую команду
ansible -i hosts.yaml web1 -m shell -a "whoami"
```

---

## Проблемы и Решения

**"[WARNING]: provided hosts list is empty"**
```bash
# Проверить синтаксис YAML
python3 -m yaml hosts.yaml

# Проверить что группа существует
ansible-inventory -i hosts.yaml --graph | grep webservers
```

**"Permission denied (publickey)"**
```bash
# Проверить ключ в инвентаре
cat hosts.yaml | grep private_key_file

# Проверить что ключ существует
ls -la ~/.ssh/id_ed25519

# Проверить права
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh
```

**Хост не в инвентаре**
```bash
# Добавить хост
ansible-inventory -i hosts.yaml --list | grep hostname

# Или добавить руками в hosts.yaml
```

---

**Следующее:** [[04-modules-execution|Модули и Выполнение]]
