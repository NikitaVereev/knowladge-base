---
title: "systemd"
type: explanation
tags: [linux, systemd, systemctl, journalctl, units, services, targets, timers]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/01-what-is-systemd.md + 02-units-services.md"
related:
  - "[[linux/explanation/process-model]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/reference/systemd-reference]]"
  - "[[linux/explanation/shutdown]]"
---

# systemd

> **TL;DR:** systemd — система инициализации (PID 1) и менеджер сервисов.
> `systemctl start/stop/enable/status` — управление сервисами.
> `journalctl -u service` — логи. Unit-файлы в `/etc/systemd/system/`.

## Что такое systemd

systemd — PID 1, первый процесс при загрузке Linux. Заменил SysVinit и Upstart. Управляет сервисами, монтированием, логированием, сетью, таймерами.

### Архитектура

```
systemd (PID 1)
├── systemctl        CLI для управления юнитами
├── journald         централизованное логирование
├── networkd         настройка сети
├── resolved         DNS
├── timedated        время и timezone
├── logind           управление сессиями
└── tmpfiles         временные файлы
```

## Units (юниты)

Unit — объект, которым управляет systemd. Описывается в `.unit`-файле.

| Тип | Расширение | Назначение |
|-----|-----------|-----------|
| **Service** | `.service` | Демон/сервис (nginx, sshd, postgresql) |
| **Timer** | `.timer` | Задача по расписанию (замена cron) |
| **Socket** | `.socket` | Активация сервиса при подключении |
| **Mount** | `.mount` | Монтирование файловой системы |
| **Target** | `.target` | Группа юнитов (аналог runlevel) |
| **Path** | `.path` | Реакция на изменение файла |
| **Slice** | `.slice` | Группировка ресурсов (cgroups) |

### Расположение файлов

| Путь | Приоритет | Описание |
|------|----------|---------|
| `/etc/systemd/system/` | Высший | Ваши и переопределённые юниты |
| `/run/systemd/system/` | Средний | Runtime (временные) |
| `/usr/lib/systemd/system/` | Низший | Юниты из пакетов (не редактировать!) |

## Структура Service-файла

```ini
[Unit]
Description=My Application
Documentation=https://example.com/docs
After=network.target postgresql.service
Requires=postgresql.service
Wants=redis.service

[Service]
Type=simple
User=app
Group=app
WorkingDirectory=/opt/myapp
Environment=NODE_ENV=production
EnvironmentFile=/opt/myapp/.env
ExecStartPre=/opt/myapp/pre-start.sh
ExecStart=/usr/bin/node /opt/myapp/server.js
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Секция [Unit]

| Параметр | Описание |
|----------|---------|
| `Description` | Человекочитаемое описание |
| `After` | Запускать **после** этих юнитов (порядок) |
| `Requires` | Зависимость: если зависимость упала, этот тоже останавливается |
| `Wants` | Мягкая зависимость: если зависимость упала, этот продолжает |

### Секция [Service]

| Параметр | Описание |
|----------|---------|
| `Type` | `simple` (default), `forking`, `oneshot`, `notify` |
| `ExecStart` | Команда запуска (обязательно абсолютный путь) |
| `ExecReload` | Команда reload (обычно `kill -HUP`) |
| `Restart` | `on-failure`, `always`, `on-abnormal`, `no` |
| `RestartSec` | Задержка перед перезапуском |
| `User/Group` | От какого пользователя запускать |
| `Environment` | Переменные окружения |
| `EnvironmentFile` | Файл с переменными |

### Секция [Install]

| Параметр | Описание |
|----------|---------|
| `WantedBy=multi-user.target` | Включить при загрузке в multi-user (стандарт для серверов) |
| `WantedBy=graphical.target` | Включить при загрузке GUI |

## Targets (цели)

Группа юнитов = состояние системы (аналог runlevel).

| Target | Аналог | Описание |
|--------|--------|---------|
| `poweroff.target` | runlevel 0 | Выключение |
| `rescue.target` | runlevel 1 | Однопользовательский (recovery) |
| `multi-user.target` | runlevel 3 | Многопользовательский (без GUI) |
| `graphical.target` | runlevel 5 | С графическим интерфейсом |
| `reboot.target` | runlevel 6 | Перезагрузка |

```bash
# Текущий target
systemctl get-default

# Сменить default
sudo systemctl set-default multi-user.target

# Переключиться сейчас
sudo systemctl isolate multi-user.target
```

## Timers (замена cron)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# Активировать таймер
sudo systemctl enable --now backup.timer

# Список таймеров
systemctl list-timers
```

## journalctl (логи)

```bash
journalctl -u nginx          # логи конкретного сервиса
journalctl -u nginx -f       # follow (реальное время)
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err   # только ошибки
journalctl -b                # с последней загрузки
journalctl --disk-usage      # сколько занимают логи
```

## Процесс загрузки

```
BIOS/UEFI → Bootloader (GRUB) → Kernel → systemd (PID 1) → default.target
                                              ↓
                                    Параллельный запуск юнитов
                                    (network, sshd, nginx...)
```

```bash
# Анализ времени загрузки
systemd-analyze
systemd-analyze blame         # самые медленные юниты
systemd-analyze critical-chain
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Отредактировали юнит — не работает | `sudo systemctl daemon-reload` после изменений |
| Сервис enabled, но не запущен | `enable` = автозапуск при boot. `start` = запустить сейчас. `enable --now` = оба |
| `ExecStart` с относительным путём | Только абсолютные пути! `/usr/bin/node`, не `node` |
| Сервис постоянно перезапускается | `journalctl -u svc -n 50` → найти причину падения |
| Логи занимают много места | `journalctl --vacuum-size=500M` |
