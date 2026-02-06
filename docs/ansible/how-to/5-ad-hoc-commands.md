---
title: "5 Ad-Hoc команды Ansible"
description: "Как запускать задачи без плейбуков: пинг, проверка аптайма, управление пакетами."
---

Ad-Hoc команды позволяют выполнить одно действие на группе серверов прямо из консоли. Это удобно для быстрых проверок.

**Синтаксис:**
```bash
ansible [группа] -i [инвентарь] -m [модуль] -a "[аргументы]"
```

## Примеры использования

### 1. Проверка доступности (Ping)
Модуль `ping` не делает ICMP пинг, он проверяет, что Python запускается и SSH работает.
```bash
ansible all -m ping
```

### 2. Выполнение Shell-команд
```bash
# Проверить uptime
ansible webservers -m shell -a "uptime"

# Проверить свободное место
ansible all -m shell -a "df -h"
```

### 3. Управление файлами (Copy)
Быстро закинуть файл на все серверы.
```bash
ansible webservers -m copy -a "src=./manual.pdf dest=/tmp/"
```

### 4. Управление пакетами (с sudo)
Флаг `-b` (become) нужен для выполнения от root (sudo).
```bash
# Установить htop
ansible webservers -m apt -a "name=htop state=present" -b
```

### 5. Сбор информации (Setup)
Посмотреть все факты о сервере.
```bash
ansible web-01 -m setup
```
