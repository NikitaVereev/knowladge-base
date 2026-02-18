---
title: "Планирование повторяющихся задач"
type: how-to
tags: [linux, cron, crontab, systemd, timers, scheduling, automation]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 7.6"
  docs: "https://man7.org/linux/man-pages/man5/crontab.5.html"
  docs2: "https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html"
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/systemd-reference]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/how-to/recipes/backup-script]]"
---

# Планирование повторяющихся задач

> **TL;DR:** Два механизма: **cron** (классический, формат `* * * * * command`) и **systemd timers** (современный, пара unit-файлов `.timer` + `.service`). cron проще для одноразовых задач, timers дают логирование через journald, зависимости и гарантию выполнения пропущенных запусков (`Persistent=true`).

## Предварительные условия

- cron: пакет `cron` (Debian/Ubuntu) или `cronie` (Arch/Fedora)
- systemd timers: systemd (есть в любом современном дистрибутиве)

## cron

Демон `cron` просыпается каждую минуту, проверяет расписания и выполняет команды от имени соответствующих пользователей.

### Формат crontab

```
┌───────────── минута (0–59)
│ ┌───────────── час (0–23)
│ │ ┌───────────── день месяца (1–31)
│ │ │ ┌───────────── месяц (1–12 или jan–dec)
│ │ │ │ ┌───────────── день недели (0–7, 0 и 7 = воскресенье, или mon–sun)
│ │ │ │ │
* * * * *  command
```

Каждое поле допускает:

| Синтаксис | Значение | Пример |
|-----------|----------|--------|
| `*` | Любое значение | `* * * * *` — каждую минуту |
| `5` | Конкретное | `5 * * * *` — в :05 каждого часа |
| `1,15` | Перечисление | `0 1,15 * * *` — в 01:00 и 15:00 |
| `1-5` | Диапазон | `0 9 * * 1-5` — будни в 09:00 |
| `*/10` | Шаг | `*/10 * * * *` — каждые 10 минут |

### Управление crontab

```bash
# Редактировать crontab текущего пользователя
crontab -e

# Показать текущий crontab
crontab -l

# Редактировать crontab другого пользователя (от root)
sudo crontab -u john -e

# Удалить crontab (осторожно — без подтверждения)
crontab -r
```

Каждый пользователь имеет свой crontab. Задачи выполняются от имени владельца crontab с его uid/gid и окружением.

### Примеры

```bash
# Бэкап БД каждый день в 03:00
0 3 * * *  /opt/scripts/backup-db.sh >> /var/log/backup.log 2>&1

# Очистка tmp каждое воскресенье в 04:30
30 4 * * 0  find /tmp -type f -mtime +7 -delete

# Проверка SSL-сертификатов дважды в день
0 9,21 * * *  /opt/scripts/check-certs.sh

# Ротация логов каждый понедельник в 00:00
0 0 * * 1  /usr/sbin/logrotate /etc/logrotate.conf

# Каждые 5 минут (мониторинг)
*/5 * * * *  /opt/scripts/healthcheck.sh
```

### Окружение cron

cron выполняет команды в **минимальном окружении** — не читает `~/.bashrc`, `~/.profile`. PATH обычно содержит только `/usr/bin:/bin`. Это главный источник проблем «работает в терминале, не работает в cron».

```bash
# Решение 1: абсолютные пути в команде
0 3 * * *  /usr/bin/python3 /opt/scripts/report.py

# Решение 2: задать PATH в crontab
PATH=/usr/local/bin:/usr/bin:/bin
0 3 * * *  python3 /opt/scripts/report.py

# Решение 3: задать переменные окружения
SHELL=/bin/bash
MAILTO=admin@example.com
0 3 * * *  /opt/scripts/backup.sh
```

`MAILTO` — cron отправляет stdout/stderr задачи на указанный email. Если `MAILTO=""` — вывод отбрасывается. Без `MAILTO` — отправляет владельцу crontab (если настроен MTA).

### Системный crontab: /etc/crontab и /etc/cron.d/

Помимо пользовательских crontab, существуют системные:

```bash
# /etc/crontab — системный crontab, содержит дополнительное поле: имя пользователя
# мин час день мес dow  user  command
  0   3   *   *   *    root  /opt/scripts/system-backup.sh

# /etc/cron.d/ — директория для drop-in файлов (формат как /etc/crontab)
# /etc/cron.daily/   — скрипты, запускаемые ежедневно
# /etc/cron.hourly/  — ежечасно
# /etc/cron.weekly/  — еженедельно
# /etc/cron.monthly/ — ежемесячно
```

Директории `cron.daily/`, `cron.hourly/` и т.д. — это обычные скрипты (без crontab-формата), которые запускает `anacron` или cron через записи в `/etc/crontab`. Скрипты должны быть исполняемыми и **без расширения** (файл `backup`, а не `backup.sh` — некоторые реализации cron игнорируют файлы с точкой в имени).

### Ограничение доступа

```bash
# /etc/cron.allow — только перечисленные пользователи могут использовать cron
# /etc/cron.deny  — запрет для перечисленных (если cron.allow не существует)

# Если оба файла отсутствуют — поведение зависит от дистрибутива:
#   Debian: все могут использовать cron
#   Red Hat: только root
```

## systemd timers

Пара unit-файлов: `.timer` (расписание) + `.service` (действие). Timer активирует соответствующий service по расписанию.

### Структура

```
/etc/systemd/system/
├── backup.timer        ← расписание
└── backup.service      ← что выполнять
```

Timer и service связываются **по имени**: `backup.timer` активирует `backup.service`. Если нужно другое имя — указать `Unit=` в секции `[Timer]`.

### Пример: ежедневный бэкап

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 03:00:00    # каждый день в 03:00
Persistent=true               # выполнить при загрузке, если пропущен
RandomizedDelaySec=600         # случайная задержка 0–10 мин (разнести нагрузку)

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Database backup
After=postgresql.service       # запускать после PostgreSQL

[Service]
Type=oneshot                   # выполнить и завершиться
User=backup                   # от какого пользователя
ExecStart=/opt/scripts/backup-db.sh
StandardOutput=journal         # логи в journald
```

```bash
# Активировать
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# Проверить
systemctl list-timers
# NEXT                        LEFT     LAST                        PASSED   UNIT
# Wed 2026-02-19 03:00:00 MSK 14h left Tue 2026-02-18 03:04:23 MSK 9h ago  backup.timer

# Логи последнего запуска
journalctl -u backup.service -n 20

# Запустить вручную (для теста)
sudo systemctl start backup.service
```

### Формат OnCalendar

```
DayOfWeek Year-Month-Day Hour:Minute:Second
```

| Значение | Эквивалент cron | Описание |
|----------|----------------|----------|
| `*-*-* 03:00:00` | `0 3 * * *` | Ежедневно в 03:00 |
| `Mon *-*-* 00:00:00` | `0 0 * * 1` | Каждый понедельник |
| `*-*-* *:00:00` | `0 * * * *` | Каждый час |
| `*-*-* *:*:00` | `* * * * *` | Каждую минуту |
| `*-*-* 09,21:00:00` | `0 9,21 * * *` | В 09:00 и 21:00 |
| `Mon..Fri *-*-* 09:00:00` | `0 9 * * 1-5` | Будни в 09:00 |
| `*-*-01 00:00:00` | `0 0 1 * *` | Первое число каждого месяца |
| `*-01,07-01 00:00:00` | `0 0 1 1,7 *` | 1 января и 1 июля |
| `quarterly` | — | Раз в квартал |
| `weekly` | `0 0 * * 0` | Еженедельно (понедельник, 00:00) |
| `daily` | `0 0 * * *` | Ежедневно в 00:00 |
| `hourly` | `0 * * * *` | Каждый час |

```bash
# Проверить расписание (покажет следующие запуски)
systemd-analyze calendar "*-*-* 03:00:00"
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
systemd-analyze calendar "weekly"
```

### Монотонные таймеры (относительные интервалы)

Вместо календарного расписания — интервал от события:

```ini
[Timer]
OnBootSec=5min              # через 5 минут после загрузки
OnUnitActiveSec=1h          # каждый час после последнего запуска service
OnStartupSec=10min          # через 10 минут после запуска systemd (user instance)
```

| Параметр | Отсчёт от |
|----------|----------|
| `OnBootSec` | Загрузка системы |
| `OnStartupSec` | Запуск user instance systemd |
| `OnUnitActiveSec` | Последняя активация service |
| `OnUnitInactiveSec` | Последняя деактивация service |
| `OnActiveSec` | Активация самого timer |

```ini
# Пример: healthcheck каждые 5 минут
[Timer]
OnBootSec=1min               # первый запуск через минуту после загрузки
OnUnitActiveSec=5min          # затем каждые 5 минут
```

### Persistent=true

Если система была выключена в момент запланированного выполнения, timer с `Persistent=true` запустит задачу при следующей загрузке. Информация о последнем запуске хранится на диске.

cron **не имеет** такой возможности — пропущенные задачи просто не выполняются (для этого в cron-мире существует `anacron`, но он работает только с daily/weekly/monthly интервалами).

## cron vs systemd timers

| | cron | systemd timers |
|---|------|---------------|
| Конфигурация | Одна строка | Два unit-файла |
| Логирование | stdout/stderr → email или `/dev/null` | journald (`journalctl -u service`) |
| Зависимости | Нет | `After=`, `Requires=`, `Wants=` |
| Пропущенный запуск | Потерян (без anacron) | `Persistent=true` |
| Точность | Минута | Микросекунда (на практике — секунда) |
| Ресурсы | Не контролирует | `MemoryMax=`, `CPUQuota=`, `Nice=` через cgroup |
| Рандомизация | Нет | `RandomizedDelaySec=` |
| Тестирование | Ждать или менять расписание | `systemctl start service` |
| Мониторинг | `crontab -l`, нет статуса выполнения | `systemctl list-timers`, `systemctl status` |
| Когда использовать | Быстрые одноразовые задачи, скрипты | Production-сервисы, задачи с зависимостями |

### Когда что выбирать

**cron** — когда нужно быстро добавить задачу: `crontab -e`, одна строка, готово. Для личных скриптов, простых задач без зависимостей.

**systemd timers** — когда важны: логирование, обработка ошибок, зависимости от других сервисов, контроль ресурсов, гарантия выполнения после простоя. Для production-серверов и инфраструктурных задач.

## Пример: миграция с cron на systemd timer

Crontab-строка:

```bash
0 3 * * * /opt/scripts/cleanup.sh >> /var/log/cleanup.log 2>&1
```

Эквивалент в systemd:

```ini
# /etc/systemd/system/cleanup.timer
[Unit]
Description=Daily cleanup

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/cleanup.service
[Unit]
Description=Cleanup old files

[Service]
Type=oneshot
ExecStart=/opt/scripts/cleanup.sh
# Логи автоматически в journald — не нужен >> file.log 2>&1
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cleanup.timer

# Убрать из cron
crontab -e  # удалить строку
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|----------|---------|---------|
| Относительный путь в cron | `command not found` | Абсолютные пути или задать `PATH=` в crontab |
| Скрипт работает в терминале, не в cron | Нет переменных окружения | cron не читает `.bashrc`. Задать переменные в crontab или в скрипте |
| `% ` в команде cron | Команда обрезается | `%` в cron — перевод строки. Экранировать: `\%` |
| Нет логов задачи cron | Не знаешь, выполнилась ли | `>> /var/log/task.log 2>&1` или перейти на systemd timer |
| Timer enabled, service не запускается | `systemctl list-timers` показывает timer | `systemctl status service` — проверить ошибки. `daemon-reload` после изменений |
| Задача пропущена (сервер был выключен) | Бэкап не создан | cron: использовать anacron. Timers: `Persistent=true` |
| Две копии задачи одновременно | Предыдущий запуск не завершился | cron: flock (`flock -n /tmp/task.lock /opt/scripts/task.sh`). Timers: service по умолчанию не запускается повторно, пока предыдущий активен |

## Связанные материалы

- [[linux/explanation/systemd]] — units, targets, timers (обзор)
- [[linux/reference/systemd-reference]] — шаблон timer/service, systemctl команды
- [[linux/how-to/manage-services]] — systemctl start/stop/enable
- [[linux/how-to/recipes/backup-script]] — готовый скрипт бэкапа