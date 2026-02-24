---
title: "Обработка текста: sed, awk, xargs"
type: reference
tags: [bash, sed, awk, xargs, find, text-processing, regex, scripting]
sources:
  docs: "https://man7.org/linux/man-pages/man1/sed.1.html"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 11.10"
related:
  - "[[bash/how-to/write-scripts]]"
  - "[[bash/explanation/shell-internals]]"
  - "[[linux/tutorials/04-shell-and-scripting]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/tutorials/03-filesystem-and-commands]]"
---

# Обработка текста: sed, awk, xargs

> **TL;DR:** `sed` — автоматическая замена и удаление строк. `awk` — извлечение полей (колонок).
> `xargs` — превратить stdin в аргументы команды. Все три работают как фильтры в конвейерах.

## sed — потоковый редактор

`sed` (stream editor) читает текст построчно, применяет операции и выводит результат в stdout. Не изменяет исходный файл (если не указать `-i`).

### Замена: `s/pattern/replacement/`

```bash
# Заменить первое вхождение в каждой строке
sed 's/old/new/' file.txt

# Заменить ВСЕ вхождения в строке (флаг g — global)
sed 's/old/new/g' file.txt

# Практические примеры
sed 's/:/%/' /etc/passwd              # первое двоеточие → %
sed 's/:/%/g' /etc/passwd             # все двоеточия → %
sed 's/^#//' config.conf              # раскомментировать (убрать # в начале)
sed 's/^/    /' code.py               # добавить отступ (4 пробела)
sed 's/[[:space:]]*$//' file.txt      # удалить trailing whitespace
```

### Удаление строк: `d`

```bash
# По номеру строки
sed '3d' file.txt                     # удалить 3-ю строку
sed '3,6d' file.txt                   # удалить строки 3–6

# По regex-паттерну
sed '/^#/d' config.conf               # удалить комментарии
sed '/^$/d' file.txt                  # удалить пустые строки
sed '/DEBUG/d' app.log                # удалить строки с DEBUG
```

### Адресация

Адрес определяет, к каким строкам применяется операция:

| Адрес | Значение | Пример |
|-------|---------|--------|
| (пусто) | Все строки | `sed 's/a/b/g'` |
| `5` | Строка 5 | `sed '5d'` |
| `3,6` | Строки 3–6 | `sed '3,6d'` |
| `$` | Последняя строка | `sed '$d'` |
| `/regex/` | Строки, соответствующие regex | `sed '/error/d'` |
| `/start/,/end/` | Диапазон между паттернами | `sed '/BEGIN/,/END/d'` |

### Полезные комбинации

```bash
# Несколько операций через -e
sed -e 's/foo/bar/g' -e '/^$/d' file.txt

# Редактирование файла на месте (-i)
sed -i 's/localhost/127.0.0.1/g' config.conf

# С бэкапом (безопаснее)
sed -i.bak 's/localhost/127.0.0.1/g' config.conf
# Создаст config.conf.bak с оригиналом

# Вывести только совпавшие строки (-n + p)
sed -n '/ERROR/p' app.log             # аналог grep, но с поддержкой адресов

# Вставить строку после совпадения
sed '/\[server\]/a bind_address = 0.0.0.0' config.ini
```

> **sed vs другие инструменты:** для простого поиска → `grep`. Для замены в одном файле → `sed`. Для сложной логики с условиями → `awk` или Python.

## awk — извлечение полей

`awk` — полноценный язык программирования, но в 90% случаев используется для одного: вытащить нужную колонку из текстового вывода. awk автоматически разбивает каждую строку на поля по пробелам/табам.

### Базовое использование: `'{print $N}'`

```bash
# Поля нумеруются с $1. $0 — вся строка
ls -l | awk '{print $5}'              # размер файла (5-е поле)
ls -l | awk '{print $9}'              # имя файла (9-е поле)
ls -l | awk '{print $5, $9}'          # размер и имя

# Практические примеры
df -h | awk '{print $5, $6}'          # % занятости и точка монтирования
ps aux | awk '{print $1, $2, $11}'    # user, PID, command
free -m | awk '/Mem:/ {print $3}'     # used memory (строка Mem, поле 3)
ip route | awk '/default/ {print $3}' # IP шлюза
```

### Разделитель полей: `-F`

```bash
# По умолчанию разделитель — пробелы/табы
# -F меняет разделитель

awk -F: '{print $1, $3}' /etc/passwd  # login и UID (разделитель :)
awk -F, '{print $2}' data.csv         # второе поле CSV
awk -F'|' '{print $1}' table.txt      # разделитель |
```

### Фильтрация строк

```bash
# Условие перед действием
awk '$3 > 1000 {print $1, $3}' /etc/passwd    # UID > 1000 (с -F:)
awk -F: '$3 > 1000 {print $1}' /etc/passwd    # пользователи с UID > 1000
awk '/error/ {print $0}' app.log               # строки с "error"
awk 'NR > 1 {print $0}' data.csv              # пропустить заголовок (строки > 1)
awk 'NF > 0' file.txt                         # непустые строки (NF = кол-во полей)
```

### Встроенные переменные

| Переменная | Значение |
|-----------|---------|
| `$0` | Вся строка |
| `$1`, `$2`, ... | Поля |
| `NF` | Количество полей в текущей строке |
| `NR` | Номер текущей строки |
| `FS` | Разделитель полей (input) |
| `OFS` | Разделитель полей (output) |

```bash
# Последнее поле (какое бы оно ни было)
awk '{print $NF}' file.txt

# Предпоследнее
awk '{print $(NF-1)}' file.txt

# Подсчёт строк (аналог wc -l)
awk 'END {print NR}' file.txt

# Сумма значений в колонке
awk '{sum += $5} END {print sum}' data.txt
```

> **awk vs Python:** для извлечения полей в CLI и простых конвейерах — awk быстрее и короче. Для сложной логики, обработки JSON/XML, работы с сетью — Python.

## xargs — stdin → аргументы команды

`xargs` читает stdin и передаёт прочитанное как аргументы указанной команде. Нужен, когда входных данных слишком много для одной командной строки, или когда нужно применить команду к каждой строке из stdin.

### Базовое использование

```bash
# Без xargs: echo передаёт текст, но не запускает команду для каждого
# С xargs: каждая строка из stdin → аргумент команды

echo -e "file1\nfile2\nfile3" | xargs rm        # rm file1 file2 file3
cat urls.txt | xargs wget                        # wget url1 url2 ...
```

### xargs + find (основной паттерн)

```bash
# НЕПРАВИЛЬНО — проблемы с пробелами в именах файлов
find . -name '*.gif' -print | xargs file

# ПРАВИЛЬНО — NULL-разделитель (всегда используйте эту форму)
find . -name '*.gif' -print0 | xargs -0 file

# Почему: -print0 разделяет файлы символом \0 (а не \n)
# xargs -0 читает по \0. Пробелы и спецсимволы в именах не ломают команду.
```

### Альтернатива: `find -exec`

```bash
# -exec запускает команду для КАЖДОГО файла отдельно
find . -name '*.gif' -exec file {} \;
#                                 ↑↑
#                        {} = имя файла, \; = конец команды

# -exec с + (группировка, быстрее)
find . -name '*.gif' -exec file {} +
# Передаёт файлы пачкой, как xargs
```

| Подход | Когда использовать |
|--------|-------------------|
| `find -print0 \| xargs -0` | Много файлов, нужна скорость |
| `find -exec {} \;` | Нужно выполнить для каждого файла отдельно |
| `find -exec {} +` | Золотая середина: группировка без xargs |

### Полезные флаги xargs

```bash
# -n N — по N аргументов за раз
echo "a b c d" | xargs -n 2 echo
# a b
# c d

# -I {} — подставить аргумент в произвольное место
cat hosts.txt | xargs -I {} ssh {} uptime
# ssh host1 uptime, ssh host2 uptime, ...

# -P N — параллельное выполнение (N процессов)
find . -name '*.jpg' -print0 | xargs -0 -P 4 -I {} convert {} -resize 50% {}
# 4 параллельных процесса конвертации

# -- — конец опций (защита от файлов, начинающихся с -)
find . -print0 | xargs -0 -- rm
```

> **Производительность:** xargs запускает отдельный процесс для каждой пачки аргументов. Для тысяч файлов это приемлемо, но не ожидайте мгновенной работы.

## Комбинирование: конвейеры на практике

```bash
# Найти 10 самых больших файлов
find /var/log -type f -exec du -b {} + | sort -rn | head -10 | awk '{print $2}'

# Заменить текст во всех конфигах
find /etc -name '*.conf' -print0 | xargs -0 sed -i 's/old_host/new_host/g'

# Убить все процессы по имени
ps aux | awk '/zombie_process/ {print $2}' | xargs kill

# Посчитать строки кода в проекте
find src/ -name '*.py' -print0 | xargs -0 wc -l | tail -1

# Извлечь уникальные IP из лога
awk '{print $1}' access.log | sort -u

# Логи: только ошибки, без timestamps
grep ERROR app.log | sed 's/^[0-9-]* [0-9:]* //' | sort -u
```

## Типичные ошибки

| Ошибка | Правильно |
|--------|-----------|
| `find \| xargs` без `-print0 -0` | `find -print0 \| xargs -0` (пробелы в именах!) |
| `sed 's/foo/bar'` (без закрывающего /) | `sed 's/foo/bar/'` |
| `awk {print $1}` (без кавычек) | `awk '{print $1}'` (shell раскроет `$1`!) |
| `sed -i` без бэкапа на проде | `sed -i.bak` или проверить через `sed ... \| head` сначала |
| `xargs rm` для файлов с `-` в начале | `xargs -- rm` или `xargs rm --` |
