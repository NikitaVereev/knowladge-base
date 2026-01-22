---
title: "4 Запуск Ansible Playbook"
description: "Команда ansible-playbook, флаги --check, --limit, --tags и уровни вербозности."
---

Основная команда для запуска сценариев — `ansible-playbook`.

**Базовый синтаксис:**
```bash
ansible-playbook -i hosts.yaml site.yml
```

## Полезные флаги

### 1. Безопасный режим (Dry Run)
Флаг `--check` (или `-C`) прогоняет сценарий без внесения изменений. Удобно для валидации перед деплоем.
```bash
ansible-playbook site.yml --check
```
*Примечание: Команды `shell`/`command` могут не работать в check mode, если они меняют состояние системы.*

### 2. Ограничение хостов (--limit)
Запустить плейбук только на части инвентаря.
```bash
# Только на web1
ansible-playbook site.yml --limit web1

# На группе db, исключая db-backup
ansible-playbook site.yml --limit "db:!db-backup"
```

### 3. Отладка (Verbosity)
Если что-то не работает, добавьте `-v`.
* `-v`: Показать результаты выполнения (stdout).
* `-vv`: Показать входные параметры модулей.
* `-vvv`: Показать информацию о подключении SSH (самый детальный).
```bash
ansible-playbook site.yml -vvv
```

### 4. Теги (--tags)
Запустить только определенные задачи.
```bash
ansible-playbook site.yml --tags "deploy,config"
```

### 5. Повышение привилегий
* `--ask-become-pass` (`-K`): Запросить пароль для sudo, если он требуется.
