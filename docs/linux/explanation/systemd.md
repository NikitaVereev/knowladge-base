---
title: "systemd"
type: explanation
tags: [linux, systemd, systemctl, journalctl, units, services, targets, timers, socket-activation, sysv-init, dependencies]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/01-what-is-systemd.md + 02-units-services.md"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 6"
related:
  - "[[linux/explanation/process-model]]"
  - "[[linux/explanation/boot-process]]"
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
| **Slice** | `.slice` | Группировка ресурсов (cgroups) → Подробнее: [[linux/explanation/cgroups]] |

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
| `Before` | Запускать **перед** этими юнитами |
| `Requires` | Зависимость: если зависимость упала, этот тоже останавливается |
| `Requisite` | Как `Requires`, но если зависимость **ещё не запущена** — сразу ошибка (не пытается запустить) |
| `Wants` | Мягкая зависимость: если зависимость упала, этот продолжает |
| `BindsTo` | Жёсткая привязка: если зависимость остановлена **по любой причине**, этот тоже |
| `Conflicts` | Взаимоисключение: запуск одного останавливает другой |
| `ConditionPathExists` | Запускать только если путь существует |

**After/Before vs Requires/Wants.** Это ортогональные понятия: `Requires/Wants` = «запустить вместе», `After/Before` = «в каком порядке». Без `After` зависимости запускаются **параллельно** (ключевое преимущество systemd над SysV init).

```bash
# Просмотр зависимостей
systemctl list-dependencies nginx.service
systemctl list-dependencies --reverse nginx.service   # кто зависит от nginx
```

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

Определяет поведение `systemctl enable/disable`.

| Параметр | Описание |
|----------|---------|
| `WantedBy=multi-user.target` | Включить при загрузке в multi-user (стандарт для серверов) |
| `WantedBy=graphical.target` | Включить при загрузке GUI |
| `RequiredBy=...` | Жёсткая зависимость в обратную сторону |

**Как работает enable/disable.** `systemctl enable nginx` создаёт символическую ссылку:

```
/etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service
```

`systemctl disable nginx` удаляет эту ссылку. При загрузке systemd обходит каталоги `.wants/` и `.requires/` каждого target и запускает найденные юниты. Таким образом `enable` — это не «запуск», а «включение автозапуска».

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

## Socket Activation

systemd может создавать сокеты **до** запуска сервиса и передавать их при первом подключении. Это позволяет:

- Запускать сервисы **по требованию** (не тратя ресурсы на неиспользуемые)
- Не терять соединения при перезапуске сервиса (systemd буферизирует)
- Избежать проблем с порядком запуска (сокет готов, даже если сервис ещё загружается)

```ini
# /etc/systemd/system/echo.socket
[Unit]
Description=Echo socket

[Socket]
ListenStream=7777
Accept=true

[Install]
WantedBy=sockets.target
```

```ini
# /etc/systemd/system/echo@.service
[Unit]
Description=Echo service

[Service]
ExecStart=/usr/bin/cat
StandardInput=socket
```

`Accept=true` — systemd создаёт отдельный экземпляр сервиса для каждого входящего соединения (шаблон `echo@.service`, `@` = инстанс).

```bash
systemctl enable --now echo.socket     # активировать сокет
systemctl list-sockets                 # все сокеты
```

## System V init (историческая справка)

До systemd большинство дистрибутивов использовали SysV init. Основные отличия:

| | SysV init | systemd |
|---|---|---|
| Конфигурация | Shell-скрипты в `/etc/init.d/` | Декларативные unit-файлы |
| Запуск сервисов | Последовательный | Параллельный |
| Зависимости | Нет (только порядок через номер: S01, S02...) | Явные (Wants, Requires, After) |
| Runlevels/Targets | 0–6 (числовые) | Именованные targets |
| Управление | `service nginx start`, `/etc/init.d/nginx start` | `systemctl start nginx` |
| Логирование | Каждый сервис сам | Централизованный journald |

SysV использовал каталоги `/etc/rc0.d/` – `/etc/rc6.d/` с символическими ссылками вида `S85nginx` → `/etc/init.d/nginx` (S = Start, число = порядок, K = Kill).

systemd сохраняет обратную совместимость: скрипты в `/etc/init.d/` автоматически получают unit-обёртки. Но новые сервисы всегда следует создавать как unit-файлы.

## Аварийная загрузка

При проблемах с загрузкой systemd предоставляет несколько аварийных режимов:

| Режим | Как попасть | Что доступно |
|---|---|---|
| **rescue.target** | `systemd.unit=rescue.target` в параметрах ядра | Минимальная система, сеть отключена, root shell |
| **emergency.target** | `systemd.unit=emergency.target` | Только корневая FS (read-only), ещё минимальнее |
| **init=/bin/bash** | Параметр ядра `init=/bin/bash` | Прямой shell без systemd, FS read-only |

```bash
# В GRUB: нажать e, добавить в строку linux:
systemd.unit=rescue.target     # мягкий вариант
init=/bin/bash                 # жёсткий вариант (без systemd)
# Загрузить: Ctrl+X
```

Подробнее об аварийной загрузке: [[linux/explanation/boot-process]].

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Отредактировали юнит — не работает | `sudo systemctl daemon-reload` после изменений |
| Сервис enabled, но не запущен | `enable` = автозапуск при boot. `start` = запустить сейчас. `enable --now` = оба |
| `ExecStart` с относительным путём | Только абсолютные пути! `/usr/bin/node`, не `node` |
| Сервис постоянно перезапускается | `journalctl -u svc -n 50` → найти причину падения |
| Логи занимают много места | `journalctl --vacuum-size=500M` |
