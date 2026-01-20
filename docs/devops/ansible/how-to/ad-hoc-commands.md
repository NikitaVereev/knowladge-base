---
title: "Ad-Hoc команды Ansible"
description: "Как запускать задачи без плейбуков: пинг, проверка аптайма, перезагрузка сервисов."
---


Ad-Hoc команды позволяют выполнить одно действие на группе серверов прямо из консоли, не создавая файл playbook.

**Синтаксис:**
```bash
ansible [группа] -i [инвентарь] -m [модуль] -a "[аргументы]"
```

## Примеры использования

### 1. Проверка доступности (Ping)
```bash
ansible all -m ping
```

### 2. Сбор информации (Setup/Facts)
Посмотреть ОС, IP-адреса, память и другие факты.
```bash
ansible webservers -m setup
# Фильтр по конкретному факту
ansible webservers -m setup -a "filter=ansible_distribution"
```

### 3. Выполнение Shell-команд
```bash
# Проверить uptime
ansible all -m shell -a "uptime"

# Проверить свободное место
ansible all -m shell -a "df -h"
```

### 4. Управление файлами (Copy)
Быстро закинуть файл на все сервера.
```bash
ansible webservers -m copy -a "src=./manual.pdf dest=/tmp/"
```

### 5. Управление пакетами (с sudo)
Флаг `-b` (become) нужен для выполнения от root (sudo).
```bash
ansible webservers -m apt -a "name=htop state=present" -b
```

### 6. Перезагрузка сервисов
```bash
ansible webservers -m systemd -a "name=nginx state=restarted" -b
```

## Полезные флаги
* `-i hosts.ini` — путь к инвентарю (если не в ansible.cfg).
* `-b` — использовать `sudo`.
* `-K` — запросить пароль для `sudo`.
* `-f 10` — выполнять в 10 потоков (по умолчанию 5).

## Связанные материалы
- [[devops/ansible/reference/common-modules|Справочник модулей]]
