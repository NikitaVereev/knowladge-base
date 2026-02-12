---
title: "Написание Bash-скриптов"
type: how-to
tags: [linux, bash, scripting, patterns, error-handling, arguments, arrays, debug]
related:
  - "[[linux/tutorials/04-shell-and-scripting]]"
  - "[[linux/how-to/recipes/backup-script]]"
  - "[[linux/reference/cheatsheet]]"
---

# Написание Bash-скриптов

> **TL;DR:** `set -euo pipefail` в начале. Кавычки вокруг переменных. `shellcheck` для проверки.
> Функции + local переменные + trap для cleanup. getopts для аргументов.

## Шаблон скрипта

```bash
#!/bin/bash
set -euo pipefail

# --- Описание ---
# script.sh — что делает скрипт
# Usage: ./script.sh [-v] [-o output] <input>

# --- Константы ---
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly VERSION="1.0.0"

# --- Функции ---
log()   { echo "[$(date '+%H:%M:%S')] $*"; }
error() { echo "[ERROR] $*" >&2; exit 1; }

usage() {
    cat << EOF
Usage: $(basename "$0") [OPTIONS] <input>

Options:
  -o FILE   Output file (default: stdout)
  -v        Verbose mode
  -h        Show help
EOF
    exit 0
}

cleanup() {
    rm -f "${TMPFILE:-}"
    log "Cleanup done"
}
trap cleanup EXIT

# --- Аргументы ---
VERBOSE=false
OUTPUT=""

while getopts "o:vh" opt; do
    case $opt in
        o) OUTPUT="$OPTARG" ;;
        v) VERBOSE=true ;;
        h) usage ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))

[[ $# -lt 1 ]] && error "Missing input argument. Use -h for help."
INPUT="$1"

# --- Основной код ---
TMPFILE=$(mktemp)
log "Processing $INPUT..."

if $VERBOSE; then
    log "Verbose mode enabled"
fi

# ... ваш код ...

log "Done!"
```

## Обработка ошибок

```bash
# set -euo pipefail — разобрать по частям:
set -e            # exit при ошибке (ненулевой код)
set -u            # exit при обращении к undefined переменной
set -o pipefail   # ошибка в пайпе → ошибка всего пайпа

# Игнорировать ошибку конкретной команды
command_that_may_fail || true

# Проверить код возврата
if ! command; then
    error "Command failed"
fi

# trap для cleanup
trap 'rm -f /tmp/tempfile.$$' EXIT    # при любом завершении
trap 'error "Interrupted"' INT TERM   # при Ctrl+C
```

## Работа с аргументами

```bash
# Позиционные
echo "Script: $0, First: $1, All: $@, Count: $#"

# Значения по умолчанию
name="${1:-World}"                 # $1 или "World"
dir="${OUTPUT_DIR:=/tmp}"          # переменная или "/tmp" (и присвоить)

# getopts (короткие опции)
while getopts "f:n:vh" opt; do
    case $opt in
        f) FILE="$OPTARG" ;;
        n) COUNT="$OPTARG" ;;
        v) VERBOSE=true ;;
        h) usage ;;
    esac
done
shift $((OPTIND - 1))             # оставшиеся — позиционные
```

## Строки

```bash
str="Hello World"
echo "${#str}"                    # длина: 11
echo "${str,,}"                   # lowercase: hello world
echo "${str^^}"                   # UPPERCASE: HELLO WORLD
echo "${str/World/Bash}"          # замена: Hello Bash
echo "${str:0:5}"                 # подстрока: Hello

# Проверки
[[ -z "$str" ]]                   # пустая?
[[ -n "$str" ]]                   # не пустая?
[[ "$str" == *World* ]]           # содержит?
[[ "$str" =~ ^Hello ]]           # regex?
```

## Массивы

```bash
# Объявление
arr=("one" "two" "three")

# Доступ
echo "${arr[0]}"                  # первый
echo "${arr[@]}"                  # все элементы
echo "${#arr[@]}"                 # длина

# Итерация
for item in "${arr[@]}"; do
    echo "$item"
done

# Добавить
arr+=("four")

# Ассоциативные (bash 4+)
declare -A config
config[host]="localhost"
config[port]="8080"
echo "${config[host]}:${config[port]}"
```

## Отладка

```bash
# Режим отладки
bash -x script.sh                 # показывать каждую команду
set -x                            # включить внутри скрипта
set +x                            # выключить

# shellcheck — статический анализ (РЕКОМЕНДУЕТСЯ)
# Установить: apt install shellcheck / pacman -S shellcheck
shellcheck script.sh
```

## Типичные ошибки

| Ошибка | Правильно |
|--------|-----------|
| `[ $var = "val" ]` | `[ "$var" = "val" ]` (кавычки!) |
| `if [ $? == 0 ]` | `if command; then` |
| `cd dir && ...` без проверки | `cd dir || exit 1` |
| Пробелы в `name = "val"` | `name="val"` (без пробелов) |
| `echo $array` | `echo "${array[@]}"` |
| Не используют shellcheck | `shellcheck script.sh` ловит 90% багов |