---
title: "Группы управления (cgroups)"
type: explanation
tags: [linux, cgroups, resources, cpu, memory, systemd, containers, namespaces, oom]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 8.6"
  docs: "https://docs.kernel.org/admin-guide/cgroup-v2.html"
related:
  - "[[linux/explanation/process-model]]"
  - "[[linux/explanation/systemd]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/how-to/monitor-system]]"
  - "[[kubernetes/how-to/resource-limits]]"
---

# Группы управления (cgroups)

> **TL;DR:** cgroups (control groups) — механизм ядра для ограничения, учёта и изоляции ресурсов (CPU, память, I/O, сеть) для групп процессов. systemd автоматически создаёт cgroup для каждого сервиса. Docker и Kubernetes используют cgroups для лимитов контейнеров. Без cgroups один процесс может сожрать всю память и положить сервер.

## Зачем это знать

В обычной Linux-системе любой процесс может потреблять столько CPU и памяти, сколько доступно. Один утёкший процесс способен исчерпать всю RAM, вызвать OOM killer и уронить критичные сервисы. cgroups решают эту проблему на уровне ядра — не доверяя приложениям самим себя ограничивать.

Понимание cgroups объясняет:

- Как `MemoryMax=512M` в systemd unit-файле реально ограничивает сервис
- Почему контейнер с `--memory=1g` не может выйти за лимит
- Откуда Kubernetes знает, сколько ресурсов потребляет Pod
- Что именно делает OOM killer и почему он убил именно этот процесс

## Что такое cgroup

cgroup — это именованная группа процессов, к которой применяются ограничения ресурсов. Это механизм ядра, представленный как виртуальная файловая система (обычно смонтирована в `/sys/fs/cgroup/`).

Ключевые свойства:

- **Иерархичность** — cgroup образуют дерево. Дочерняя группа не может превысить лимиты родительской
- **Наследование** — дочерний процесс (fork) попадает в cgroup родителя
- **Учёт** — ядро отслеживает потребление ресурсов каждой группой
- **Принудительность** — лимиты enforced ядром, процесс не может их обойти

## cgroups v1 vs v2

### cgroups v1 (устаревшая)

Каждый контроллер ресурсов (cpu, memory, blkio) — отдельная иерархия. Процесс мог быть в разных cgroup для разных контроллеров. Это создавало сложности: непоследовательное поведение, проблемы с взаимодействием контроллеров.

```
/sys/fs/cgroup/
├── cpu/             ← отдельная иерархия
│   └── myapp/
├── memory/          ← отдельная иерархия
│   └── myapp/
└── blkio/           ← отдельная иерархия
    └── myapp/
```

### cgroups v2 (актуальная)

Единая иерархия для всех контроллеров. Процесс принадлежит ровно одной cgroup, и все контроллеры применяются к ней.

```
/sys/fs/cgroup/
├── cgroup.controllers        # доступные контроллеры
├── cgroup.subtree_control    # активные контроллеры для дочерних
├── system.slice/             # системные сервисы
│   ├── nginx.service/
│   │   ├── cgroup.procs      # PID процессов в группе
│   │   ├── memory.current    # текущее потребление памяти
│   │   ├── memory.max        # лимит памяти
│   │   └── cpu.weight        # вес CPU
│   └── postgresql.service/
├── user.slice/               # пользовательские сессии
│   └── user-1000.slice/
└── init.scope/               # PID 1 (systemd)
```

cgroups v2 — стандарт начиная с: Fedora 31+, Ubuntu 21.10+, Debian 11+, Arch Linux (с 2021). Проверить:

```bash
# Какая версия используется
stat -fc %T /sys/fs/cgroup/
# cgroup2fs → v2
# tmpfs     → v1 (или гибрид)

# Или
mount | grep cgroup
```

## Контроллеры

Контроллер — компонент ядра, управляющий конкретным типом ресурса. Каждый контроллер предоставляет файлы-интерфейсы внутри cgroup.

### memory — контроллер памяти

| Файл | Описание |
|------|----------|
| `memory.current` | Текущее потребление (байты) |
| `memory.min` | Гарантированный минимум (ядро не отберёт) |
| `memory.low` | Мягкий минимум (ядро старается не отбирать) |
| `memory.high` | Мягкий лимит (при превышении — throttling, замедление) |
| `memory.max` | Жёсткий лимит (при превышении — OOM kill) |
| `memory.peak` | Максимальное потребление за время жизни |
| `memory.swap.max` | Лимит swap |

```bash
# Посмотреть потребление памяти сервисом nginx
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
# 52428800  (50 МБ)

cat /sys/fs/cgroup/system.slice/nginx.service/memory.max
# max  (без ограничений, если не задан MemoryMax= в unit)
```

Когда процесс пытается выделить память сверх `memory.max`, ядро вызывает **OOM killer** — он завершает процесс(ы) внутри этой cgroup. Именно cgroup, а не всей системы — это ключевое отличие от системного OOM.

### cpu — контроллер процессора

| Файл | Описание |
|------|----------|
| `cpu.weight` | Относительный вес (1–10000, default 100) |
| `cpu.max` | Жёсткий лимит: `$QUOTA $PERIOD` (микросекунды) |
| `cpu.stat` | Статистика: usage_usec, user_usec, system_usec |
| `cpu.pressure` | PSI (Pressure Stall Information) |

```bash
# cpu.weight — пропорциональное распределение
# Если у группы A weight=100, у B weight=200:
# B получает в 2 раза больше CPU, когда обе конкурируют
# Когда конкуренции нет — обе могут использовать 100% CPU

# cpu.max — жёсткий лимит
# "200000 1000000" → максимум 200мс из каждой 1000мс → 20% CPU
echo "200000 1000000" > /sys/fs/cgroup/myapp/cpu.max
```

Важное различие: `cpu.weight` — это **относительный** вес, работает только при конкуренции. `cpu.max` — **абсолютный** лимит, процесс не получит больше даже если CPU свободен.

### io — контроллер блочного ввода-вывода

| Файл | Описание |
|------|----------|
| `io.weight` | Относительный вес I/O (1–10000) |
| `io.max` | Лимит IOPS и bandwidth per device |
| `io.stat` | Статистика по устройствам |

```bash
# Ограничить запись на sda: max 10 МБ/с
echo "8:0 wbps=10485760" > /sys/fs/cgroup/myapp/io.max
```

### pids — контроллер количества процессов

| Файл | Описание |
|------|----------|
| `pids.max` | Максимум процессов в группе |
| `pids.current` | Текущее количество |

Защита от fork-бомб: процесс не может создать бесконечное количество дочерних.

## Как systemd использует cgroups

systemd — основной потребитель cgroups на современном Linux. При запуске каждого сервиса systemd автоматически создаёт cgroup и помещает в неё процессы.

### Иерархия slice → scope/service

```
-.slice (корневая)
├── system.slice                     # системные сервисы
│   ├── nginx.service
│   ├── postgresql.service
│   └── sshd.service
├── user.slice                       # пользовательские сессии
│   ├── user-1000.slice
│   │   └── session-1.scope          # SSH-сессия
│   └── user-1001.slice
└── machine.slice                    # виртуальные машины, контейнеры
    └── docker-abc123.scope
```

| Unit-тип | Описание |
|----------|----------|
| `.slice` | Группировка (узел дерева, не содержит процессов напрямую) |
| `.service` | Сервис (ExecStart создаёт процессы внутри cgroup) |
| `.scope` | Внешне созданные процессы (сессии, контейнеры) |

### Директивы ресурсов в unit-файлах

```ini
[Service]
# Память
MemoryMin=128M           # гарантированный минимум → memory.min
MemoryLow=256M           # мягкий минимум → memory.low
MemoryHigh=768M          # мягкий лимит (throttling) → memory.high
MemoryMax=1G             # жёсткий лимит (OOM kill) → memory.max
MemorySwapMax=0          # запретить swap → memory.swap.max

# CPU
CPUWeight=200            # относительный вес → cpu.weight
CPUQuota=50%             # жёсткий лимит (50% одного ядра) → cpu.max

# I/O
IOWeight=100             # относительный вес → io.weight

# PID
TasksMax=512             # максимум процессов/потоков → pids.max
```

```bash
# Посмотреть cgroup сервиса
systemctl show nginx.service -p ControlGroup
# ControlGroup=/system.slice/nginx.service

# Ресурсы сервиса
systemctl status nginx.service
# └─12345 nginx: master process
#   Memory: 48.2M (max: 1.0G)
#   CPU: 1.234s

# Все cgroup в системе
systemd-cgls

# Потребление по cgroup (аналог top для cgroup)
systemd-cgtop
```

## Практический пример: ограничение сервиса

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application

[Service]
Type=simple
User=app
ExecStart=/opt/myapp/server

# Лимит памяти: при 768M начнёт тормозить, при 1G — OOM kill
MemoryHigh=768M
MemoryMax=1G
MemorySwapMax=0

# Не больше 80% одного ядра CPU
CPUQuota=80%

# Не больше 100 процессов/потоков (защита от fork-бомб)
TasksMax=100

# Ограничить доступ к файловой системе
ProtectSystem=strict
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp

# Проверить лимиты
cat /sys/fs/cgroup/system.slice/myapp.service/memory.max
# 1073741824  (1G в байтах)

cat /sys/fs/cgroup/system.slice/myapp.service/cpu.max
# 80000 100000  (80мс из 100мс = 80%)
```

## Как Docker и Kubernetes используют cgroups

Контейнеры — это cgroups + namespaces. cgroups ограничивают **сколько** ресурсов видит контейнер, namespaces — **какие** ресурсы видит.

```bash
# Docker: --memory и --cpus транслируются в cgroup
docker run --memory=512m --cpus=1.5 nginx

# Найти cgroup контейнера
docker inspect --format='{{.HostConfig.CgroupParent}}' <container>

# Kubernetes: resources.limits → cgroup контейнера
# spec:
#   containers:
#     - resources:
#         limits:
#           memory: "512Mi"   → memory.max
#           cpu: "1500m"      → cpu.max
#         requests:
#           memory: "256Mi"   → (влияет на scheduling, не на cgroup напрямую)
#           cpu: "500m"       → cpu.weight (пропорционально)
```

Подробнее о requests/limits в Kubernetes: [[kubernetes/how-to/resource-limits]].

## Мониторинг cgroups

```bash
# Дерево cgroup
systemd-cgls

# Top по cgroup (CPU, memory, I/O в реальном времени)
systemd-cgtop

# Потребление конкретного сервиса
systemctl status nginx.service       # краткая сводка
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.stat

# PSI (Pressure Stall Information) — насколько ресурсы «под давлением»
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.pressure
# some avg10=0.00 avg60=0.00 avg300=0.00 total=12345
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# OOM events
cat /sys/fs/cgroup/system.slice/myapp.service/memory.events
# oom 3        ← OOM killer сработал 3 раза
# oom_kill 3
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Процесс убит OOM killer | `dmesg \| grep -i oom`, exit code 137 | Увеличить `MemoryMax` или найти утечку памяти в приложении |
| CPU throttling, приложение тормозит | `cpu.stat` показывает `nr_throttled` > 0 | Увеличить `CPUQuota` или оптимизировать приложение |
| `MemoryHigh` вместо `MemoryMax` | Приложение тормозит, но не убивается | Это штатно: `MemoryHigh` = throttling. Для жёсткого лимита — `MemoryMax` |
| Лимиты не работают | cgroups v1 на старом ядре | Проверить `stat -fc %T /sys/fs/cgroup/`. Некоторые контроллеры недоступны в v1 |
| Fork-бомба кладёт сервер | Один пользователь создаёт тысячи процессов | `TasksMax=` в unit-файле или `UserTasksMax=` в `/etc/systemd/logind.conf` |
| Контейнер потребляет больше лимита | `docker stats` показывает превышение | Проверить что cgroups v2 активен, лимит задан корректно |

## Связанные материалы

- [[linux/explanation/process-model]] — PID, состояния процессов, сигналы
- [[linux/explanation/systemd]] — units, slices, targets
- [[linux/how-to/manage-services]] — systemctl, unit-файлы
- [[linux/how-to/monitor-system]] — top, htop, iostat, journalctl
- [[kubernetes/how-to/resource-limits]] — requests/limits в Kubernetes (используют cgroups)
