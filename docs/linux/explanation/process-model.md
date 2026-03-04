---
title: "Процессы и сервисы"
type: explanation
tags: [linux, processes, signals, pid, daemon, background, foreground, priority, monitoring, threads, strace, lsof, vmstat]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 8"
related:
  - "[[linux/explanation/architecture]]"
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/explanation/shutdown]]"
  - "[[linux/explanation/permissions-model]]"
  - "[[linux/explanation/cgroups]]"
---

# Процессы и сервисы

> **TL;DR:** Процесс = запущенная программа с PID. Daemon = фоновый сервис.
> `ps aux` — список, `kill PID` — завершить, `systemctl` — управление сервисами.
> Сигналы: SIGTERM (15, мягко) → SIGKILL (9, принудительно).

## Что такое процесс

Каждая запущенная программа — процесс с уникальным PID (Process ID). Процесс имеет владельца, приоритет, используемые ресурсы (CPU, RAM), открытые файлы и сетевые соединения.

Каждый процесс создан другим процессом (parent). Первый процесс — `init` (PID 1), в современных системах это systemd.

```bash
# Дерево процессов
pstree
# systemd─┬─sshd───sshd───bash───vim
#          ├─nginx───nginx
#          └─postgres───postgres
```

### Как создаются процессы: fork() и exec()

Все процессы в Linux (кроме init) создаются через два системных вызова:

- **`fork()`** — ядро создаёт **почти идентичную копию** текущего процесса (тот же код, те же переменные, те же open files). Копия получает новый PID
- **`exec(program)`** — ядро **заменяет** текущий процесс на новую программу. Старый код исчезает, загружается `program`

Как это работает на практике — когда вы вводите `ls` в терминале:

```
bash (PID 100)
    │
    ├─ fork() → bash-копия (PID 101)   # ядро клонирует bash
    │               │
    │               └─ exec(ls)          # копия заменяется на ls
    │                    │
    │                    └─ ls работает, выводит результат, завершается
    │
    └─ bash ждёт (waitpid) → получает код возврата ls → показывает prompt
```

Именно поэтому переменные, заданные в дочернем скрипте, не видны в родительском shell — fork создаёт **копию**, а не общее пространство. Подробнее: [[bash/explanation/shell-internals]].

```bash
# Увидеть fork/exec в реальном времени
strace -f -e trace=clone,execve bash -c 'ls /tmp' 2>&1 | grep -E 'clone|execve'
# execve("/bin/bash", ...)         # bash запущен
# clone(...)  = 12345              # fork → дочерний PID 12345
# [pid 12345] execve("/bin/ls", ...) # дочерний процесс стал ls
```

## Просмотр процессов

```bash
# Все процессы (самая частая команда)
ps aux
# USER  PID  %CPU  %MEM  VSZ   RSS   TTY  STAT  START  TIME  COMMAND
# root  1    0.0   0.1   168   11M   ?    Ss    Jan01  0:05  /sbin/init

# Только свои
ps ux

# По имени
pgrep -l nginx
pidof nginx

# Интерактивный мониторинг
top                        # базовый
htop                       # улучшенный (нужно установить)
```

### Состояния процессов (STAT)

| Код | Состояние | Описание |
|-----|----------|---------|
| `R` | Running | Выполняется или готов |
| `S` | Sleeping | Ожидает события (I/O, сигнал) |
| `D` | Disk sleep | Ожидает I/O (нельзя убить!) |
| `Z` | Zombie | Завершился, но parent не прочитал статус |
| `T` | Stopped | Остановлен (Ctrl+Z) |

## Управление процессами

### Foreground и Background

```bash
# Запустить в фоне
sleep 100 &
# [1] 12345

# Показать фоновые задачи
jobs
# [1]+  Running  sleep 100 &

# Перевести в фон (Ctrl+Z, потом bg)
vim file.txt
# Ctrl+Z → Stopped
bg                         # продолжить в фоне

# Вернуть в foreground
fg %1

# Запустить и отвязать от терминала
nohup long-task.sh &
# продолжит работать после закрытия терминала
```

### Сигналы

```bash
# Мягкое завершение (SIGTERM = 15)
kill PID
kill -15 PID

# Принудительное завершение (SIGKILL = 9)
kill -9 PID

# По имени
killall nginx
pkill -f "python app.py"

# Перечитать конфигурацию (SIGHUP = 1)
kill -HUP PID
```

| Сигнал | Номер | Действие | Перехватываемый |
|--------|-------|---------|----------------|
| `SIGTERM` | 15 | Мягкое завершение | Да (default kill) |
| `SIGKILL` | 9 | Принудительное | **Нет** (крайний случай) |
| `SIGHUP` | 1 | Перечитать конфиг | Да |
| `SIGINT` | 2 | Прервать (Ctrl+C) | Да |
| `SIGSTOP` | 19 | Остановить | **Нет** |
| `SIGCONT` | 18 | Продолжить | Да |

### Приоритет (nice)

```bash
# Запустить с низким приоритетом (19 = самый низкий)
nice -n 19 make -j4

# Изменить приоритет работающего процесса
renice -n 10 -p PID

# Диапазон: -20 (highest) до 19 (lowest)
# Только root может ставить отрицательный nice
```

## Daemons (фоновые сервисы)

Daemon — процесс, работающий в фоне без терминала. Обычно управляется через systemd. Примеры: `sshd` (SSH), `nginx` (веб-сервер), `postgresql` (БД), `cron` (задачи по расписанию).

Подробнее: [[linux/explanation/systemd]].

## Мониторинг процессов

### top и htop

`top` — основной интерактивный монитор. Обновляется каждые несколько секунд.

```bash
top
# Горячие клавиши внутри top:
# P — сортировать по CPU
# M — сортировать по памяти
# T — сортировать по времени CPU
# u — фильтр по пользователю
# f — выбрать столбцы
# H — показать потоки (threads)
# k — убить процесс (ввести PID)
# q — выйти
```

`htop` — улучшенная версия: цветной, поддержка мыши, горизонтальный скролл, дерево процессов (F5).

### lsof — открытые файлы и соединения

«Всё — файл» → `lsof` показывает всё, с чем работает процесс.

```bash
# Все открытые файлы процесса
lsof -p PID
# COMMAND  PID  USER  FD    TYPE  DEVICE  SIZE/OFF  NODE  NAME
# nginx    5678 www   0u    IPv4  12345   0t0       TCP   *:80 (LISTEN)
# nginx    5678 www   4r    REG   8,1     4096      1234  /var/log/nginx/access.log

# Кто использует файл или каталог
lsof /var/log/syslog
lsof +D /var/log/              # рекурсивно в каталоге

# Кто слушает/использует порт
lsof -i :80
lsof -i TCP:22

# Кто использует сетевые соединения к конкретному хосту
lsof -i @192.168.1.1
```

Поле `FD`: `r` = read, `w` = write, `u` = read/write, число — файловый дескриптор (0 = stdin, 1 = stdout, 2 = stderr).

### strace и ltrace — трассировка вызовов

```bash
# Трассировка системных вызовов (ядро)
strace ls /tmp                   # запустить с трассировкой
strace -p PID                    # подключиться к работающему процессу
strace -f -p PID                 # включая дочерние процессы (fork)
strace -e trace=open,read ls     # только конкретные syscalls
strace -e trace=network curl ... # только сетевые вызовы
strace -c ls /tmp                # статистика: какие syscalls, сколько раз, время

# Трассировка библиотечных вызовов (userspace)
ltrace ./myapp                   # какие функции libc вызываются
```

> `strace` незаменим для отладки: «почему сервис не находит конфиг?» → `strace -e trace=open,openat` покажет какие файлы он пытается открыть.

## Потоки (threads)

Поток — единица выполнения внутри процесса. Все потоки одного процесса разделяют память и файловые дескрипторы, но имеют собственный стек и Thread ID (TID).

```bash
# Показать потоки процесса
ps m -p PID                    # m = show threads
ps -eLf | grep nginx           # L = show LWP (lightweight process = thread)

# В top
top -H                         # показать потоки вместо процессов
# или нажать H внутри top

# Кол-во потоков процесса
ls /proc/PID/task/ | wc -l
cat /proc/PID/status | grep Threads
```

Однопоточный процесс (bash, sed) имеет PID = TID. Многопоточный (nginx, java) имеет один PID и несколько TID. Преимущества потоков перед отдельными процессами: разделяемая память (быстрый обмен данными) и меньший overhead при создании.

## Мониторинг ресурсов

### Load average

```bash
uptime
# 12:00  up 30 days,  load average: 1.5, 0.8, 0.6
#                                    1m   5m   15m
```

Load average — среднее число процессов в состоянии R (running) или D (disk sleep). Если load > nproc — система перегружена. Рост от 1 мин к 15 мин — нагрузка снижается. Рост от 15 мин к 1 мин — нагрузка растёт.

> D-state (uninterruptible sleep) — процесс ждёт завершения дискового I/O. Такие процессы нельзя убить сигналом (даже `kill -9`), потому что ядро не может прервать операцию I/O. Высокий load при низком CPU usage = проблема с дисковой подсистемой.

### Ключевые инструменты

| Инструмент | Что показывает | Когда использовать |
|---|---|---|
| `top` / `htop` | CPU, RAM по процессам в реальном времени | «Что грузит систему?» |
| `vmstat 1` | CPU, память, swap, I/O одним взглядом | «Общая картина: CPU/RAM/диск?» |
| `iostat -x 1` | Дисковый I/O по устройствам | «Диск перегружен?» |
| `iotop` | Дисковый I/O по процессам | «Кто пишет на диск?» |
| `pidstat -p PID 1` | CPU/RAM/IO конкретного процесса | «Чем занят этот PID?» |
| `lsof -p PID` | Открытые файлы и сокеты процесса | «Что он открыл?» |
| `strace -p PID` | Системные вызовы процесса | «Почему он завис/падает?» |

Практические рецепты и примеры вывода: [[linux/how-to/monitor-system]].

## Полезные утилиты

```bash
# Что использует порт 8080?
lsof -i :8080
ss -tlnp | grep 8080

# Что использует файл?
lsof /var/log/syslog
fuser /var/log/syslog

# Трассировка
strace -e trace=network -p PID   # сетевые syscalls процесса
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Процесс не реагирует на `kill` | Попробовать `kill -9`. Если D-state — ждать завершения I/O |
| Zombie-процессы | Не занимают ресурсы. Убить parent-процесс для очистки |
| Нагрузка высокая, но CPU idle | Много D-state процессов → проблема с диском (I/O wait) |
| Порт «занят» после остановки | `ss -tlnp | grep PORT` → найти процесс → `kill PID` |
| Процесс умирает при закрытии SSH | Использовать `nohup`, `tmux` или `screen` |
## Связанные материалы

- [[linux/explanation/systemd]] — управление сервисами, PID 1
- [[linux/explanation/cgroups]] — ограничение ресурсов процессов
- [[linux/how-to/manage-services]] — systemctl start/stop/enable
- [[linux/how-to/monitor-system]] — практический мониторинг: CPU, RAM, диск
- [[linux/reference/cheatsheet]] — справочник команд
