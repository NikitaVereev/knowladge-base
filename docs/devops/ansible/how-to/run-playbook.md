---
title: "Запуск Ansible Playbook"
description: "Команда ansible-playbook, флаги --check, --limit, --tags и уровни вербозности."
---


Основная команда для запуска сценариев — `ansible-playbook`.

**Базовый синтаксис:**
```bash
ansible-playbook -i hosts.yaml site.yml
```

## Полезные флаги

### 1. Безопасный режим (Dry Run)
Флаг `--check` (или `-C`) прогоняет сценарий без внесения изменений. Удобно для валидации.
```bash
ansible-playbook site.yml --check
```
> **Важно:** Некоторые модули (например, shell) могут не поддерживать check mode или вести себя некорректно, если зависят от результатов предыдущих шагов.

### 2. Ограничение хостов
Флаг `--limit` (или `-l`) позволяет запустить плейбук только на части инвентаря.
```bash
# Только на web1
ansible-playbook site.yml --limit web1

# Только на группе db, исключая db-backup
ansible-playbook site.yml --limit "db:!db-backup"
```

### 3. Отладка (Verbosity)
Если что-то не работает, добавьте `-v`.
* `-v`: Показать результаты выполнения.
* `-vv`: Показать входные/выходные данные.
* `-vvv`: Информация о подключении SSH (самый детальный).
```bash
ansible-playbook site.yml -vvv
```

### 4. Запрос паролей
* `-K` (`--ask-become-pass`): Запросить пароль для sudo.
* `-k` (`--ask-pass`): Запросить пароль SSH (если не по ключам).

### 5. Запуск с определенного места
Если плейбук упал на середине, можно продолжить:
```bash
ansible-playbook site.yml --start-at-task="Install Nginx"
```

## Связанные материалы
- [[devops/ansible/explanation/playbook-basics|Структура Playbook]]
