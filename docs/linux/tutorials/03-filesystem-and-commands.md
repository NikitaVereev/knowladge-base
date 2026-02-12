---
title: "03 — Файловая система и команды"
type: tutorial
tags: [linux, tutorial, filesystem, commands, navigation, files, pipes, redirect]
related:
  - "[[linux/tutorials/02-package-management]]"
  - "[[linux/tutorials/04-shell-and-scripting]]"
  - "[[linux/explanation/filesystem]]"
  - "[[linux/reference/filesystem-hierarchy]]"
---

# Tutorial 03 — Файловая система и команды

> **Цель:** Уверенная навигация по файловой системе. Создание, копирование, поиск файлов.
> Перенаправление, пайпы, основные текстовые утилиты.

**Время:** ~30 минут
**Требования:** Терминал Linux.

## Шаг 1. Навигация

```bash
# Где я?
pwd
# /home/user

# Перейти
cd /etc                    # абсолютный путь
cd ~                       # домашняя
cd ..                      # на уровень выше
cd -                       # предыдущая директория

# Что здесь?
ls                         # список
ls -la                     # подробный + скрытые
ls -lh                     # с размерами в human-readable
ls -lt                     # по времени (новые первые)
```

## Шаг 2. Работа с файлами

```bash
# Создать
touch file.txt             # пустой файл
mkdir -p projects/web      # директория (с родительскими)
echo "hello" > file.txt    # записать в файл (перезаписать)
echo "world" >> file.txt   # дописать в конец

# Копировать / Переместить / Удалить
cp file.txt backup.txt
cp -r projects/ backup/    # директорию рекурсивно
mv file.txt renamed.txt    # переименовать / переместить
rm file.txt                # удалить файл
rm -rf old_dir/            # удалить директорию (⚠️ без подтверждения!)

# Ссылки
ln -s /path/to/target link_name  # символическая ссылка
```

## Шаг 3. Просмотр содержимого

```bash
cat file.txt               # весь файл
head -20 file.txt          # первые 20 строк
tail -20 file.txt          # последние 20
tail -f /var/log/syslog    # следить в реальном времени
less file.txt              # постраничный (q — выход, / — поиск)
wc -l file.txt             # количество строк
file image.png             # определить тип файла
```

## Шаг 4. Поиск

```bash
# По имени
find /etc -name "*.conf"
find /home -name "*.py" -type f

# По размеру
find / -size +100M 2>/dev/null

# По времени (изменён за последние 7 дней)
find /var/log -mtime -7

# Быстрый поиск (по индексу)
locate nginx.conf          # нужен mlocate/plocate
sudo updatedb              # обновить индекс

# Поиск текста в файлах
grep "error" /var/log/syslog
grep -r "TODO" ~/projects/ # рекурсивно
grep -rn "pattern" .       # с номерами строк
grep -i "error" file.txt   # без учёта регистра
```

## Шаг 5. Перенаправление и пайпы

```bash
# Перенаправление
command > file.txt          # stdout → файл (перезаписать)
command >> file.txt         # stdout → файл (дописать)
command 2> errors.txt       # stderr → файл
command &> all.txt          # stdout + stderr → файл
command < input.txt         # stdin ← файл

# Пайпы (|) — выход одной команды → вход другой
cat /var/log/syslog | grep error | wc -l
ps aux | grep nginx
ls -la | sort -k5 -rn      # отсортировать по размеру (5й столбец)

# Практические примеры
history | grep "apt"        # что я устанавливал?
du -sh /* 2>/dev/null | sort -rh | head -5   # топ-5 директорий по размеру
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head  # топ IP
```

## Шаг 6. Полезные текстовые утилиты

```bash
# sort — сортировка
sort file.txt               # по алфавиту
sort -n file.txt            # по числам
sort -rn file.txt           # по убыванию чисел

# uniq — уникальные строки (данные ДОЛЖНЫ быть отсортированы)
sort file.txt | uniq         # убрать дубликаты
sort file.txt | uniq -c     # с подсчётом

# cut — извлечь колонку
cut -d: -f1 /etc/passwd     # первое поле (имена пользователей)
echo "a,b,c" | cut -d, -f2  # → b

# awk — обработка колонок
ps aux | awk '{print $1, $11}'  # пользователь + команда
df -h | awk 'NR>1 {print $5, $6}'  # % использования + mount point

# sed — замена текста
sed 's/old/new/g' file.txt   # заменить (вывод в stdout)
sed -i 's/old/new/g' file.txt  # заменить в файле

# xargs — передать результат как аргументы
find . -name "*.tmp" | xargs rm
find . -name "*.py" | xargs grep "import"
```

## Шаг 7. Практика

```bash
# Создайте структуру
mkdir -p ~/practice/{docs,scripts,data}
echo "Hello World" > ~/practice/docs/readme.txt
echo -e "banana\napple\ncherry\napple" > ~/practice/data/fruits.txt

# Задания:
# 1. Посчитать уникальные фрукты
sort ~/practice/data/fruits.txt | uniq -c

# 2. Найти все .txt файлы в practice/
find ~/practice -name "*.txt"

# 3. Заменить "Hello" на "Hi" в readme.txt
sed -i 's/Hello/Hi/' ~/practice/docs/readme.txt
cat ~/practice/docs/readme.txt

# 4. Дерево директорий
tree ~/practice   # или find ~/practice -type f
```

## Что мы изучили

| Категория | Команды |
|-----------|---------|
| Навигация | `pwd`, `cd`, `ls` |
| Файлы | `touch`, `mkdir`, `cp`, `mv`, `rm`, `ln` |
| Просмотр | `cat`, `head`, `tail`, `less`, `wc` |
| Поиск | `find`, `locate`, `grep` |
| Пайпы | `|`, `>`, `>>`, `2>` |
| Текст | `sort`, `uniq`, `cut`, `awk`, `sed`, `xargs` |

## Что дальше

→ [[linux/tutorials/04-shell-and-scripting]] — bash-скрипты и автоматизация
