---
title: "Процессы и сервисы"
type: explanation
tags: [linux, processes, signals, pid, daemon, background, foreground, priority]
sources:
  original: "_inbox/01-linux/01-core-concepts/03-processes-and-services.md"
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/explanation/shutdown]]"
  - "[[linux/explanation/permissions-model]]"
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

## Полезные утилиты

```bash
# Что использует порт 8080?
lsof -i :8080
ss -tlnp | grep 8080

# Что использует файл?
lsof /var/log/syslog
fuser /var/log/syslog

# Мониторинг I/O
iotop                      # кто читает/пишет на диск

# Load average
uptime
# 12:00  up 30 days,  load average: 0.5, 0.7, 0.8
# load avg за 1, 5, 15 минут. > кол-ва ядер = перегрузка
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Процесс не реагирует на `kill` | Попробовать `kill -9`. Если D-state — ждать завершения I/O |
| Zombie-процессы | Не занимают ресурсы. Убить parent-процесс для очистки |
| Нагрузка высокая, но CPU idle | Много D-state процессов → проблема с диском (I/O wait) |
| Порт «занят» после остановки | `ss -tlnp | grep PORT` → найти процесс → `kill PID` |
| Процесс умирает при закрытии SSH | Использовать `nohup`, `tmux` или `screen` |