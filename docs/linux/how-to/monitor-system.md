---
title: "Мониторинг системы"
type: how-to
tags: [linux, monitoring, top, htop, free, df, du, iostat, journalctl, load]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/05-system-monitoring.md"
related:
  - "[[linux/explanation/process-model]]"
  - "[[linux/explanation/logging]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/reference/common-errors]]"
---

# Мониторинг системы

> **TL;DR:** `htop` — CPU/RAM в реальном времени. `df -h` — диск. `free -h` — память.
> `journalctl -p err` — ошибки. Load average > кол-ва ядер = перегрузка.

## CPU

```bash
# Интерактивный мониторинг
top                                # базовый (q — выход)
htop                               # улучшенный (F6 — сортировка)

# Load average
uptime
# 12:00 up 30 days, load average: 1.2, 0.8, 0.6
#                                  1m   5m   15m
# load > nproc = перегрузка

# Количество ядер
nproc
cat /proc/cpuinfo | grep processor | wc -l

# CPU по процессам
ps aux --sort=-%cpu | head -10

# Анализ загрузки (нужен sysstat)
mpstat 1 5                         # по ядрам, каждую секунду, 5 раз
```

## Память (RAM)

```bash
free -h
#               total   used   free   shared  buff/cache  available
# Mem:          16Gi    4.2Gi  8.1Gi  256Mi   3.7Gi       11Gi
# Swap:         2.0Gi   0B     2.0Gi

# available — реально доступная (free + buff/cache, который можно отдать)

# Топ потребителей RAM
ps aux --sort=-%mem | head -10

# Детально
cat /proc/meminfo
```

## Диск

```bash
# Свободное место по разделам
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda2       100G   45G   50G  48% /

# Размер конкретной директории
du -sh /var/log/
du -sh /home/*

# Что занимает больше всего (рекурсивно)
du -sh /* 2>/dev/null | sort -rh | head -10

# I/O (нужен sysstat)
iostat -x 1 5              # дисковая нагрузка каждую секунду

# Интерактивный I/O
iotop                      # кто читает/пишет (нужен root)
```

## Сеть

```bash
# Активные соединения
ss -tlnp                   # listening TCP ports
ss -tunp                   # все TCP/UDP соединения

# Какой процесс на порту
sudo lsof -i :8080
sudo ss -tlnp | grep 8080

# Трафик (нужен nethogs)
sudo nethogs               # трафик по процессам

# Статистика интерфейсов
ip -s link
```

## Логи (journalctl)

```bash
# Ошибки с последней загрузки
journalctl -b -p err

# Конкретный сервис
journalctl -u nginx -f             # follow
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since today

# По приоритету
journalctl -p emerg                # 0 — emergency
journalctl -p alert                # 1
journalctl -p crit                 # 2
journalctl -p err                  # 3
journalctl -p warning              # 4

# Размер журнала
journalctl --disk-usage
sudo journalctl --vacuum-size=500M
```

## Анализ загрузки

```bash
# Время загрузки
systemd-analyze

# Самые медленные сервисы
systemd-analyze blame

# Цепочка зависимостей
systemd-analyze critical-chain

# Проблемные сервисы
systemctl --failed
```

## Ключевые метрики

| Метрика | Команда | Норма | Проблема |
|---------|---------|-------|---------|
| Load average | `uptime` | < nproc | > nproc × 2 |
| RAM available | `free -h` | > 20% total | < 10% |
| Disk usage | `df -h` | < 80% | > 90% |
| Swap usage | `free -h` | 0 | > 50% (мало RAM) |
| Failed services | `systemctl --failed` | 0 | > 0 |
| I/O wait | `top` (wa%) | < 5% | > 20% |

## Продвинутый мониторинг

### vmstat — CPU, память, I/O одним взглядом

```bash
vmstat 1 5               # обновлять каждую 1 сек, 5 раз
# procs ----memory---- --swap-- ---io--- -system-- ----cpu----
#  r  b  swpd  free    si  so   bi   bo   in   cs  us sy id wa
#  1  0     0  4096M    0   0    5   10  200  500  10  3 85  2
```

| Колонка | Значение | На что смотреть |
|---|---|---|
| `r` | Процессы в очереди на CPU | > nproc = перегрузка |
| `b` | Процессы в D-state (blocked I/O) | > 0 постоянно = проблема с диском |
| `si/so` | Swap in / swap out | > 0 постоянно = не хватает RAM |
| `wa` | CPU: I/O wait % | > 10% = узкое место в дисковой подсистеме |

### pidstat — статистика по конкретному процессу

```bash
# Требует пакет sysstat
pidstat -p PID 1              # CPU каждую секунду
pidstat -r -p PID 1           # память
pidstat -d -p PID 1           # дисковый I/O
```

### lsof — кто использует файл/порт

```bash
lsof -p PID                   # все файлы процесса
lsof +D /var/log/             # кто использует файлы в каталоге
lsof -i :8080                 # кто слушает порт
lsof /var/log/syslog          # кто держит файл открытым
```

### strace — трассировка системных вызовов

Показывает что процесс «просит» у ядра — для диагностики «почему не работает».

```bash
strace command                 # трассировать запуск
strace -p PID                  # подключиться к работающему процессу
strace -e trace=open,read -p PID  # только файловые операции
strace -c command              # сводка: какие syscalls сколько времени заняли
```

## Типичные ошибки

| Ситуация | Совет |
|----------|-------|
| «Сервер тормозит» | `htop` → сортировка по CPU → найти прожорливый процесс |
| «Диск заполнен» | `du -sh /* | sort -rh` → обычно `/var/log` или `/var/cache` |
| «Много RAM занято» | Проверить `available`, не `free`. Linux использует свободную RAM под кэш |
| Load average высокий, CPU idle | I/O wait → проблема с диском, не CPU |