---
title: "04 — Shell и скрипты"
type: tutorial
tags: [linux, tutorial, bash, scripting, shell, variables, loops, conditions, functions]
related:
  - "[[linux/tutorials/03-filesystem-and-commands]]"
  - "[[linux/tutorials/05-networking-basics]]"
  - "[[linux/how-to/write-bash-scripts]]"
---

# Tutorial 04 — Shell и скрипты

> **Цель:** Написать первый bash-скрипт. Переменные, условия, циклы, функции.
> Автоматизировать рутинные задачи.

**Время:** ~35 минут
**Требования:** Терминал Linux.

## Шаг 1. Первый скрипт

```bash
# Создать файл
cat > ~/hello.sh << 'EOF'
#!/bin/bash
# Мой первый скрипт

echo "Hello, $(whoami)!"
echo "Сегодня: $(date '+%Y-%m-%d %H:%M')"
echo "Вы в: $(pwd)"
EOF

# Сделать исполняемым
chmod +x ~/hello.sh

# Запустить
./hello.sh
```

`#!/bin/bash` — shebang, указывает какой интерпретатор использовать.

## Шаг 2. Переменные

```bash
#!/bin/bash

# Присвоение (БЕЗ пробелов вокруг =)
name="World"
count=42

# Использование
echo "Hello, $name!"
echo "Count: ${count} items"

# Подстановка команд
current_date=$(date +%Y%m%d)
files_count=$(ls | wc -l)
echo "Дата: $current_date, файлов: $files_count"

# Специальные переменные
echo "Скрипт: $0"           # имя скрипта
echo "Аргумент 1: $1"       # первый аргумент
echo "Все аргументы: $@"
echo "Кол-во аргументов: $#"
echo "Код выхода предыдущей: $?"
echo "PID скрипта: $$"
```

## Шаг 3. Условия (if)

```bash
#!/bin/bash

# Числа
if [ "$1" -gt 10 ]; then
    echo "Больше 10"
elif [ "$1" -eq 10 ]; then
    echo "Ровно 10"
else
    echo "Меньше 10"
fi

# Строки
if [ -z "$1" ]; then
    echo "Аргумент пустой"
fi

if [ "$USER" = "root" ]; then
    echo "Вы root!"
fi

# Файлы
if [ -f "/etc/hosts" ]; then
    echo "Файл существует"
fi

if [ -d "/home/$USER" ]; then
    echo "Директория существует"
fi
```

### Операторы сравнения

| Числа | Строки | Файлы |
|-------|--------|-------|
| `-eq` равно | `=` равны | `-f` файл существует |
| `-ne` не равно | `!=` не равны | `-d` директория существует |
| `-gt` больше | `-z` пустая | `-r` можно читать |
| `-lt` меньше | `-n` не пустая | `-w` можно писать |
| `-ge` ≥ | | `-x` можно выполнять |
| `-le` ≤ | | `-s` размер > 0 |

## Шаг 4. Циклы

```bash
#!/bin/bash

# for — список
for fruit in apple banana cherry; do
    echo "Фрукт: $fruit"
done

# for — файлы
for f in /etc/*.conf; do
    echo "Конфиг: $f ($(wc -l < "$f") строк)"
done

# for — диапазон
for i in {1..5}; do
    echo "Шаг $i"
done

# for — C-style
for ((i=0; i<10; i++)); do
    echo $i
done

# while
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# while — чтение файла построчно
while IFS= read -r line; do
    echo "Строка: $line"
done < /etc/hosts
```

## Шаг 5. Функции

```bash
#!/bin/bash

# Определение
greet() {
    local name="$1"    # local — локальная переменная
    echo "Hello, ${name:-stranger}!"  # default value
}

# Вызов
greet "Alice"
greet             # → "Hello, stranger!"

# Функция с return
is_root() {
    [ "$(id -u)" -eq 0 ]
}

if is_root; then
    echo "Running as root"
else
    echo "Not root"
fi

# Практическая функция
log() {
    echo "[$(date '+%H:%M:%S')] $*"
}

log "Starting backup..."
log "Done!"
```

## Шаг 6. Практический скрипт

```bash
#!/bin/bash
# backup-home.sh — бэкап домашней директории

set -euo pipefail          # остановиться при ошибке

# --- Настройки ---
BACKUP_DIR="/tmp/backups"
DATE=$(date +%Y%m%d_%H%M)
ARCHIVE="${BACKUP_DIR}/home-${DATE}.tar.gz"

# --- Функции ---
log() { echo "[$(date '+%H:%M:%S')] $*"; }

check_space() {
    local available
    available=$(df /tmp --output=avail | tail -1)
    if [ "$available" -lt 1048576 ]; then
        log "ERROR: Недостаточно места (<1GB)"
        exit 1
    fi
}

# --- Основной код ---
log "Запуск бэкапа..."

mkdir -p "$BACKUP_DIR"
check_space

tar czf "$ARCHIVE" \
    --exclude='.cache' \
    --exclude='node_modules' \
    --exclude='.local/share/Trash' \
    "$HOME" 2>/dev/null

SIZE=$(du -sh "$ARCHIVE" | cut -f1)
log "Бэкап создан: $ARCHIVE ($SIZE)"

# Удалить старые (старше 7 дней)
find "$BACKUP_DIR" -name "home-*.tar.gz" -mtime +7 -delete
log "Готово!"
```

```bash
chmod +x backup-home.sh
./backup-home.sh
```

## Шаг 7. Полезные паттерны

```bash
# set -euo pipefail — всегда в начале скриптов
set -e            # остановиться при ошибке
set -u            # ошибка при неопределённой переменной
set -o pipefail   # ошибка в пайпе = ошибка скрипта

# Значения по умолчанию
name="${1:-default}"       # $1 или "default"
dir="${BACKUP_DIR:=/tmp}"  # BACKUP_DIR или "/tmp" (и присвоить)

# Проверка аргументов
if [ $# -lt 1 ]; then
    echo "Usage: $0 <filename>"
    exit 1
fi

# trap — действие при завершении
cleanup() { rm -f /tmp/tempfile.$$; }
trap cleanup EXIT
```

## Типичные ошибки

| Ошибка | Правильно |
|--------|-----------|
| `name = "value"` (пробелы) | `name="value"` |
| `if [ $var = "" ]` (без кавычек) | `if [ "$var" = "" ]` |
| Забыли `chmod +x` | `chmod +x script.sh` |
| `echo` внутри `$()` — лишний newline | Используйте `printf` |

## Что дальше

→ [[linux/tutorials/05-networking-basics]] — сеть в Linux
→ [[linux/how-to/write-bash-scripts]] — продвинутые паттерны