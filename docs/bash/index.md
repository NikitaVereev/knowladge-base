---
title: "Bash"
type: index
tags: [bash, shell, scripting, sed, awk, xargs]
---

# Bash

Командная оболочка и язык сценариев. Склеивает команды в конвейеры, автоматизирует системные задачи, управляет процессами и окружением.

## Explanation

| Документ | Описание |
|----------|----------|
| [[bash/explanation/shell-environment]] | Переменные, PATH, dotfiles, readline |
| [[bash/explanation/shell-language]] | Кавычки, подстановки, glob, коды возврата — как shell разбирает команду |
| [[bash/explanation/shell-internals]] | Подоболочки, sourcing, exec — три модели запуска кода и их последствия |

## How-to

| Документ | Описание |
|----------|----------|
| [[bash/how-to/write-scripts]] | Шаблон скрипта, обработка ошибок, аргументы, mktemp, here-documents, отладка |

## Reference

| Документ | Описание |
|----------|----------|
| [[bash/reference/text-processing]] | sed (замена, удаление), awk (извлечение полей), xargs (stdin → аргументы) |

## Начать с нуля

Если вы только знакомитесь с shell — начните с пошагового tutorial:
→ [[linux/tutorials/04-shell-and-scripting]] — переменные, условия, циклы, функции, первый скрипт

## Быстрый старт

```bash
# Первый скрипт
cat > hello.sh << 'EOF'
#!/bin/bash
set -euo pipefail
echo "Hello, $(whoami)! Today is $(date +%A)."
EOF
chmod +x hello.sh
./hello.sh

# Полезные однострочники
ls -l | awk '{print $5, $9}'              # размер и имя файлов
find . -name '*.log' -print0 | xargs -0 wc -l   # строк в логах
sed -i 's/old/new/g' config.conf          # замена в файле
```

## Связанные разделы

- [[linux/index]] — Linux (shell работает в контексте ОС)
- [[linux/how-to/recipes/backup-script]] — готовый скрипт бэкапа
- [[linux/how-to/schedule-tasks]] — cron и systemd timers для запуска скриптов
- [[ssh/index]] — SSH (удалённое выполнение команд)
