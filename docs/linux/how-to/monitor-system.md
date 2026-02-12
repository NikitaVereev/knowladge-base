---
title: "Мониторинг системы"
type: how-to
tags: [linux, monitoring, top, htop, free, df, du, iostat, journalctl, load]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/05-system-monitoring.md"
related:
  - "[[linux/explanation/process-model]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/how-to/recipes/backup-script]]"
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

## Типичные ошибки

| Ситуация | Совет |
|----------|-------|
| «Сервер тормозит» | `htop` → сортировка по CPU → найти прожорливый процесс |
| «Диск заполнен» | `du -sh /* | sort -rh` → обычно `/var/log` или `/var/cache` |
| «Много RAM занято» | Проверить `available`, не `free`. Linux использует свободную RAM под кэш |
| Load average высокий, CPU idle | I/O wait → проблема с диском, не CPU |