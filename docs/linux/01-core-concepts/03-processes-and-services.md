# Processes and Services

## Overview

Процессы - это запущенные программы, работающие в Linux. Понимание как они работают критично для администрирования и решения проблем.

**Что вы узнаете:**
- Что такое процесс и как он идентифицируется
- Просмотр процессов (ps, top, htop)
- Управление процессами (kill, signals)
- systemd - система управления сервисами
- Сигналы и их использование
- Демоны и фоновые процессы
- Решение проблем с зависшими программами

## Prerequisites

Перед этим разделом нужно:
- Понимание файловой системы ([[01-filesystem-hierarchy|Filesystem Hierarchy]])
- Знание пользователей и прав ([[02-users-groups-permissions|Users and Permissions]])
- Базовое знание команд терминала

## What is a Process?

**Процесс** — запущенная программа с собственным PID (Process ID).

**PID (Process ID)** — уникальный номер процесса.

**PPID (Parent Process ID)** — PID процесса, который запустил данный процесс.

**Сигнал** — способ взаимодействия между процессами и ядром.

## Viewing Processes

### ps - Process Status

```bash
ps                          # процессы текущего пользователя в текущем терминале
ps aux                      # все процессы в системе
ps aux | grep firefox       # найти конкретный процесс
ps -ef --forest             # дерево процессов (родитель-потомок)
ps -p 1234                  # информация о конкретном PID
ps -u username              # процессы пользователя
ps -o pid,ppid,cmd          # выбранные колонки
```

**Главные колонки в ps aux:**
```
USER    - пользователь, запустивший процесс
PID     - ID процесса
%CPU    - использование CPU (0-100% на одном ядре)
%MEM    - использование памяти (% от RAM)
VSZ     - виртуальная память (KB)
RSS     - физическая память (KB)
STAT    - состояние (S=sleeping, R=running, Z=zombie, T=stopped)
START   - когда процесс запустился
TIME    - CPU время
COMMAND - команда которая запустила процесс
```

### top - Real-time Monitoring

Интерактивное отслеживание процессов в реальном времени:

```bash
top                         # основная команда
top -u username             # процессы конкретного пользователя
top -p 1234                 # конкретный PID
```

**Команды внутри top:**
```
q                 выход
k                 убить процесс (потребует PID)
h                 справка
M                 отсортировать по памяти
P                 отсортировать по CPU
T                 отсортировать по времени
```

### htop - Improved top

Красивая и удобная версия top:

```bash
htop                        # вместо top (требует установки)
htop -u username            # процессы пользователя
htop -p 1234                # конкретный процесс
```

**Команды в htop:**
```
↑↓←→              навигация
K                 убить процесс
P                 отсортировать по CPU
M                 отсортировать по памяти
q                 выход
h                 справка
```

### pgrep - Find by Name

```bash
pgrep firefox               # найти PID по названию программы
pgrep -l firefox            # с названием команды в выводе
pgrep -u username           # процессы конкретного пользователя
pgrep -a pattern            # с полной командной строкой
```

### pidof - Get PID

```bash
pidof firefox               # PID по названию программы
pidof -s firefox            # только одного процесса
```

## Process Management

### Background and Foreground

```bash
command &                   # запустить в фоне
jobs                        # список фоновых процессов
fg %1                       # вернуть job 1 на передний план
bg %1                       # продолжить job 1 в фоне
Ctrl+Z                      # приостановить текущий процесс
```

**Пример:**
```bash
# Запустили программу
long_running_program
# Нажали Ctrl+Z - она остановлена
# Запустим в фоне
bg %1
# Продолжит работать, пока мы работаем в терминале
```

### Kill - Terminate Processes

```bash
kill PID                    # SIGTERM (15) - мягкое завершение
kill -9 PID                 # SIGKILL (9) - принудительное завершение
kill -l                     # список всех сигналов
kill -TERM PID              # то же что просто kill
kill -HUP PID               # SIGHUP (1) - перезагрузка (для демонов)
killall firefox             # убить все процессы по названию
pkill -f "python script"    # убить по регулярному выражению
```

### Signals - Inter-Process Communication

**Сигналы** - это способ общения с процессами:

| Число | Название | Эффект |
|-------|----------|--------|
| 1 | SIGHUP | Hangup / перезагрузка (для демонов) |
| 2 | SIGINT | Interrupt (Ctrl+C) |
| 3 | SIGQUIT | Quit (Ctrl+\\) |
| 9 | SIGKILL | Kill (принудительное завершение) |
| 15 | SIGTERM | Terminate (мягкое завершение, по умолчанию) |
| 19 | SIGSTOP | Stop (приостановить) |
| 18 | SIGCONT | Continue (возобновить) |

**Как использовать:**
```bash
kill -SIGTERM PID           # мягко просим завершиться
sleep 2
kill -SIGKILL PID           # если не помогло, принудительно
```

### Process Priority - nice and renice

Каждый процесс имеет приоритет (-20 высший, +20 низший):

```bash
nice -n 10 command          # запустить с низким приоритетом
nice -n -10 command         # запустить с высоким приоритетом (требует sudo)
renice -n 5 -p PID          # изменить приоритет существующего процесса
renice -n -5 -p PID -u user # изменить для пользователя
```

**Пример:**
```bash
# Фоновая работа не должна замедлять систему
nice -n 15 find / -name "*.bak" -delete

# Критичный процесс нужен высокий приоритет
sudo nice -n -5 important_server
```

## systemd - Service Management

**systemd** — современная система управления процессами и сервисами (init система).

**Сервис (Unit)** — файл описывающий программу которая управляется systemd.

### systemctl - Control Services

```bash
systemctl list-units         # список всех unit'ов
systemctl list-units --all   # включая неактивные
systemctl list-units --failed # только fail'ы

systemctl status service     # статус сервиса (показывает logé)
systemctl start service      # запустить сервис
systemctl stop service       # остановить сервис
systemctl restart service    # остановить и запустить
systemctl reload service     # перезагрузить конфиг без остановки
systemctl enable service     # автозапуск при загрузке
systemctl disable service    # отключить автозапуск
systemctl enable --now service  # включить И запустить сейчас

systemctl is-active service  # активен ли? (выход 0=yes, 3=no)
systemctl is-enabled service # включен ли автозапуск?
```

**Примеры:**
```bash
sudo systemctl start nginx              # запустить nginx
sudo systemctl enable nginx             # при каждой загрузке
sudo systemctl status nginx             # статус
sudo systemctl restart apache2          # перезагрузить
sudo systemctl reload ssh               # перезагрузить конфиг
```

### Service Files

**Расположение:**
```
/etc/systemd/system/        # сервисы системы (редактируем здесь)
/usr/lib/systemd/system/    # встроенные сервисы (не трогаем)
/run/systemd/system/        # runtime сервисы
```

**Пример сервис-файла:**
```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myuser
ExecStart=/usr/local/bin/myapp
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Создание собственного сервиса:**
```bash
# 1. Создать файл
sudo nano /etc/systemd/system/myservice.service
# (скопировать пример выше)

# 2. Перезагрузить systemd
sudo systemctl daemon-reload

# 3. Включить и запустить
sudo systemctl enable --now myservice

# 4. Проверить
sudo systemctl status myservice
```

### journalctl - View Logs

```bash
journalctl                      # все логи (пагинированно)
journalctl -u service           # логи конкретного сервиса
journalctl -u service -f        # в реальном времени (follow)
journalctl -n 50                # последние 50 строк
journalctl --since "1 hour ago" # за последний час
journalctl --until "1 min ago"  # до минуты назад
journalctl -b                   # с последней загрузки
journalctl -p err               # только ошибки
journalctl --no-pager           # без постраничного просмотра
journalctl -S "2026-01-05"      # с конкретной даты
```

**Примеры:**
```bash
sudo journalctl -u nginx -f     # смотреть nginx логи в реальном времени
journalctl -p err -n 20         # последние 20 ошибок
journalctl -u ssh --since today # logи ssh за сегодня
```

## Daemons

**Демон (Daemon)** — процесс работающий в фоне без взаимодействия с пользователем.

**Типичные демоны:**
- `sshd` — SSH сервер
- `nginx`, `apache2` — веб-серверы
- `postgresql`, `mysql` — базы данных
- `rsyslog`, `journald` — логирование
- `cron` — scheduler для задач
- `cupsd` — печать
- `bluetooth` — bluetooth

Демоны обычно:
- Запускаются при загрузке системы
- Управляются через systemctl
- Логируют в /var/log или journald
- Имеют конфиги в /etc

## Troubleshooting

### Процесс не отвечает

```bash
# Способ 1: мягкое завершение
kill -TERM PID
sleep 2

# Способ 2: если не помогло, принудительно
kill -9 PID

# Или проще
killall -9 program_name
```

### Найти процесс, занимающий много CPU

```bash
ps aux | head -1 && ps aux | sort -k3 -rn | head -n 5
# Или:
top
# (нажмите P для сортировки по CPU)
```

### Найти процесс, занимающий много памяти

```bash
ps aux | head -1 && ps aux | sort -k4 -rn | head -n 5
# Или:
htop
# (нажмите M для сортировки по памяти)
```

### Сервис не запускается

```bash
# Проверить ошибку
sudo systemctl status service

# Посмотреть логи
sudo journalctl -u service -n 50

# Проверить синтаксис сервис-файла
sudo systemd-analyze verify /etc/systemd/system/service.service

# Перезагрузить конфиги
sudo systemctl daemon-reload
sudo systemctl restart service
```

### Что автоматически запускается?

```bash
# Все сервисы с автозапуском
systemctl list-unit-files --state=enabled

# Если что-то лишнее, отключить:
sudo systemctl disable service
```

### Порт занят другим процессом

```bash
# Узнать какой процесс занимает порт
sudo lsof -i :8080
sudo netstat -tlnp | grep 8080

# Убить процесс
kill -9 PID
```

## Cheat Sheet

```bash
# Просмотр процессов
ps aux                      # все процессы
ps aux | grep firefox       # найти конкретный
top                         # real-time мониторинг
htop                        # красивый real-time
pgrep firefox               # найти PID по названию

# Управление
kill PID                    # завершить
kill -9 PID                 # принудительно завершить
nice -n 10 command          # с низким приоритетом
command &                   # в фоне
jobs                        # фоновые процессы
fg %1                       # на передний план

# Сервисы
sudo systemctl start service    # запустить
sudo systemctl stop service     # остановить
sudo systemctl restart service  # перезагрузить
sudo systemctl enable service   # автозапуск
sudo systemctl status service   # статус
sudo journalctl -u service -f   # логи в реальном времени
```

## Key Takeaways

- **PID** — уникальный ID процесса
- **ps aux** — просмотреть все процессы
- **top/htop** — real-time мониторинг
- **kill TERM** — мягкое завершение
- **kill -9** — принудительное (последняя мера)
- **systemctl** — управление сервисами
- **journalctl** — просмотр логов
- **Демоны** — фоновые сервисы, управляются systemd

## Related

Следующий шаг:
- [[04-package-management-overview|Package Management]] — установка ПО

Предыдущий шаг:
- [[02-users-groups-permissions|Users and Permissions]] — управление доступом

Контекст:
- [[docs/linux/01-core-concepts/README|Core Concepts Index]] — полный индекс этого раздела

## See Also

Встроенная справка:
- `man ps` — справка по ps
- `man kill` — справка по kill
- `man systemctl` — справка по systemctl
- `man systemd` — справка по systemd
- `man journalctl` — справка по journalctl

Онлайн ресурсы:
- [systemd documentation](https://systemd.io/) — официальная документация
- [Understanding systemd](https://wiki.archlinux.org/title/systemd) — Arch Wiki
