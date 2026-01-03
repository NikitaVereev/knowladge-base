---
created: 2026-01-04
updated: 2026-01-04
tags: [linux, systemd, system-services, timers, automation, reference]
type: reference
---

# Systemd: Продвинутое управление сервисами и таймерами

## Основная идея

**systemd-basics** - это основы управления сервисами. Этот файл про **продвинутое: создание собственных service файлов, таймеры для автоматизации, зависимости между сервисами**.

**Для чего это нужно:**
- Создавать свои сервисы (приложения, скрипты)
- Автоматизировать задачи (backup'ы, очистка, мониторинг) через таймеры
- Управлять зависимостями (сервис A зависит от сервиса B)
- Восстанавливать сервисы автоматически (restart policies)
- Логировать и мониторить свои приложения

**Критичные навыки:**
- Создание .service файлов (Unit файлы)
- Создание .timer файлов (расписание как cron, но лучше)
- Environment переменные и переменные конфига
- Зависимости (Requires, Wants, Before, After)
- Auto-restart при падении

---

## ЧАСТЬ 1: Структура Unit файла (service)

### Базовая структура

```ini
[Unit]
Description=My Custom Service
Documentation=https://example.com/docs
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/my-app
Restart=on-failure
User=myuser
Group=mygroup

[Install]
WantedBy=multi-user.target
```

### Раздел [Unit]

```ini
[Unit]
Description=Description of the service
Documentation=file:///path/to/docs or https://url
After=network.target
Requires=other.service
Wants=optional.service
Conflicts=conflicting.service

# Пояснения:
# Description = что это за сервис (показывается в systemctl status)
# Documentation = ссылка на документацию
# After = запустить ПОСЛЕ этого сервиса
# Before = запустить ДО этого сервиса
# Requires = ОБЯЗАТЕЛЬНО нужен другой сервис (если он упадёт, упадёшь и ты)
# Wants = хочешь другой сервис (но не критично если упадёт)
# Conflicts = конфликтует с этим сервисом (не может работать вместе)
```

### Раздел [Service]

```ini
[Service]
Type=simple
ExecStart=/path/to/executable
ExecStop=/path/to/stop-script
Restart=on-failure
RestartSec=10
User=serviceuser
Group=servicegroup
WorkingDirectory=/home/serviceuser
Environment="VAR1=value1"
EnvironmentFile=/etc/myapp/config.env
StandardOutput=journal
StandardError=journal

# Пояснения:
# Type = тип сервиса:
#   simple = просто запускает процесс (по умолчанию)
#   forking = старый стиль, сам себя fork'ит
#   oneshot = запускается один раз и завершается
#   notify = уведомляет systemd когда готов
#   dbus = появляется на D-Bus

# ExecStart = как запустить сервис
# ExecStop = как остановить (если не стандартно)
# Restart = перезапускать ли при падении:
#   always = всегда перезапускать
#   on-failure = только при ошибке (exit code != 0)
#   on-abnormal = при abnormal exit
#   no = никогда (по умолчанию)
# RestartSec = сколько секунд ждать перед перезапуском

# User/Group = от какого пользователя запускать
# WorkingDirectory = рабочая папка
# Environment = переменные окружения
# EnvironmentFile = файл с переменными

# StandardOutput/StandardError = куда писать логи
#   journal = в journalctl (обычно используется)
#   file:/path = в файл
#   stdout/stderr = в stdout/stderr
```

### Раздел [Install]

```ini
[Install]
WantedBy=multi-user.target
RequiredBy=other.service
Alias=my-service.service

# Пояснения:
# WantedBy = включить сервис для этого target'а
#   multi-user.target = обычная загрузка (большинство сервисов)
#   graphical.target = с GUI
# RequiredBy = другой сервис требует этого
# Alias = другое имя для сервиса (можно вызывать по aliases)
```

---

## ЧАСТЬ 2: Создание своего service файла

### Место для файлов

```bash
# Системные сервисы (root, пакеты):
/etc/systemd/system/
/usr/lib/systemd/system/

# Пользовательские сервисы (твоих):
~/.config/systemd/user/

# Примеры:
/etc/systemd/system/my-app.service
~/.config/systemd/user/backup.service
```

### Пример 1: Простой сервис (Python app)

```ini
# /etc/systemd/system/my-app.service

[Unit]
Description=My Python Application
Documentation=https://github.com/user/my-app
After=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/opt/my-app
ExecStart=/usr/bin/python3 /opt/my-app/app.py
Restart=on-failure
RestartSec=10

# Логи пишутся в journalctl
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Использование:**
```bash
# Включить сервис
sudo systemctl enable my-app.service

# Запустить
sudo systemctl start my-app.service

# Проверить статус
sudo systemctl status my-app.service

# Посмотреть логи
journalctl -u my-app.service -f

# Остановить
sudo systemctl stop my-app.service
```

### Пример 2: Сервис с конфигом

```ini
# /etc/systemd/system/web-server.service

[Unit]
Description=Custom Web Server
Documentation=https://myserver.local
After=network.target mysql.service
Requires=mysql.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/myapp
Environment="ENVIRONMENT=production"
EnvironmentFile=/etc/myapp/config.env
ExecStart=/usr/local/bin/web-server --config /etc/myapp/config.yaml

Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Конфиг файл** `/etc/myapp/config.env`:
```bash
DATABASE_URL=mysql://user:pass@localhost/dbname
LOG_LEVEL=info
PORT=8000
```

### Пример 3: Oneshot сервис (запускается один раз)

```ini
# /etc/systemd/system/init-app.service

[Unit]
Description=Initialize Application
Before=my-app.service

[Service]
Type=oneshot
ExecStart=/opt/my-app/init.sh
User=appuser
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## ЧАСТЬ 3: Таймеры systemd (вместо cron)

### Концепция таймеров

```bash
# cron (старо):
# * * * * * /path/to/script
# Проблемы: не логируется, сложный синтаксис, не интегрируется с systemd

# systemd timers (ново):
# my-task.timer + my-task.service
# Преимущества: логируется в journalctl, простой синтаксис, интеграция с systemd
```

### Структура таймера

```ini
[Unit]
Description=Description of the timer
Documentation=https://example.com

[Timer]
OnBootSec=5min          # 5 минут после загрузки
OnUnitActiveSec=1hour   # Каждый час после срабатывания

Unit=my-task.service    # Какой service запускать

[Install]
WantedBy=timers.target
```

### Пример 1: Backup таймер (каждый день в 2:00)

**Файл сервиса:** `/etc/systemd/system/backup.service`
```ini
[Unit]
Description=Daily Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
StandardOutput=journal
StandardError=journal
```

**Файл таймера:** `/etc/systemd/system/backup.timer`
```ini
[Unit]
Description=Daily Backup Timer
Documentation=https://example.com/backup

[Timer]
# Запускать в 2:00 AM каждый день
OnCalendar=*-*-* 02:00:00

Unit=backup.service

[Install]
WantedBy=timers.target
```

**Использование:**
```bash
# Включить таймер
sudo systemctl enable backup.timer

# Запустить таймер
sudo systemctl start backup.timer

# Посмотреть статус таймера
sudo systemctl status backup.timer

# Посмотреть когда следующий запуск
systemctl list-timers backup.timer

# Посмотреть логи
journalctl -u backup.service -n 50

# Протестировать (запустить вручную)
sudo systemctl start backup.service
```

### Пример 2: Очистка логов (каждый день в 3:00)

```ini
# /etc/systemd/system/cleanup-logs.timer

[Unit]
Description=Cleanup Old Logs

[Timer]
OnCalendar=*-*-* 03:00:00
Unit=cleanup-logs.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/cleanup-logs.service

[Unit]
Description=Clean up old log files

[Service]
Type=oneshot
ExecStart=/usr/bin/journalctl --vacuum-time=30d
ExecStart=/usr/bin/find /var/log -name "*.log" -mtime +30 -delete
StandardOutput=journal
```

### Пример 3: Проверка здоровья (каждые 5 минут)

```ini
# /etc/systemd/system/health-check.timer

[Unit]
Description=Health Check Timer

[Timer]
# Запускать каждые 5 минут
OnBootSec=1min
OnUnitActiveSec=5min

Unit=health-check.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/health-check.service

[Unit]
Description=System Health Check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/health-check.sh
StandardOutput=journal
StandardError=journal
```

### Синтаксис OnCalendar (расписание)

```bash
# Формат: DayOfWeek Year-Month-Day Hour:Minute:Second

# Примеры:
*-*-* 02:00:00        # Каждый день в 2:00 AM
Mon *-*-* 09:00:00    # Каждый понедельник в 9:00 AM
*-*-1 00:00:00        # 1-го числа каждого месяца в полночь
*-1-1 00:00:00        # 1 января каждый год в полночь
*-*-* *:0/15:00       # Каждые 15 минут
*-*-* *:*:0/30        # Каждые 30 секунд

# Дни недели: Mon, Tue, Wed, Thu, Fri, Sat, Sun
# Месяцы: 1-12
# Дни: 1-31
# Часы: 0-23
# Минуты: 0-59
# Секунды: 0-59
# * = любое значение
# /n = каждый n-й (например 0/15 = 0, 15, 30, 45)
```

---

## ЧАСТЬ 4: Зависимости и порядок запуска

### Before и After

```ini
# Service A должен запуститься ПОСЛЕ Service B
[Unit]
Description=Service A
After=service-b.service

# Service A должен запуститься ДО Service B
[Unit]
Description=Service A
Before=service-b.service
```

### Requires vs Wants

```ini
# REQUIRES (обязательна!):
[Unit]
Requires=database.service
# Если database.service упадёт, упадёшь и ты!
# Если database.service не запустится, ты не запустишься!

# WANTS (желательна, но не критична):
[Unit]
Wants=logging.service
# Если logging.service упадёт, ты будешь работать (без логирования)
# Если logging.service не запустится, ты запустишься (но без логирования)
```

### Пример: Web app зависит от БД и Redis

```ini
# /etc/systemd/system/web-app.service

[Unit]
Description=Web Application
Documentation=https://myapp.local

# Запускать после этих сервисов
After=network.target mysql.service redis.service

# Требует БД (обязательно!)
Requires=mysql.service

# Хочет Redis (опционально)
Wants=redis.service

[Service]
Type=simple
ExecStart=/opt/web-app/run.sh
Restart=on-failure
RestartSec=10
User=www-data

[Install]
WantedBy=multi-user.target
```

---

## ЧАСТЬ 5: Переменные окружения и конфиги

### Environment в service файле

```ini
[Service]
Environment="DATABASE_URL=postgresql://localhost/mydb"
Environment="LOG_LEVEL=debug"
Environment="SECRET_KEY=mysecret"

ExecStart=/usr/bin/python3 /opt/app/app.py
```

### EnvironmentFile (отделить конфиг)

**Файл конфига** `/etc/myapp/app.env`:
```bash
DATABASE_URL=postgresql://localhost/mydb
LOG_LEVEL=info
SECRET_KEY=production-secret
API_PORT=8000
```

**Service файл:**
```ini
[Service]
EnvironmentFile=/etc/myapp/app.env

ExecStart=/usr/bin/python3 /opt/app/app.py
```

**Преимущества:**
- Конфиг отделён от service файла
- Легче менять конфиг (не нужно editировать service)
- Безопаснее (права доступа 600 для конфига)

### Пример: конфиг для backup

**Файл** `/etc/backup/backup.env`:
```bash
BACKUP_SOURCE=/home
BACKUP_DEST=/mnt/backup
BACKUP_RETENTION_DAYS=30
NOTIFICATION_EMAIL=admin@example.com
```

**Service файл:**
```ini
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
EnvironmentFile=/etc/backup/backup.env
ExecStart=/usr/local/bin/backup.sh \
  --source $BACKUP_SOURCE \
  --dest $BACKUP_DEST \
  --retention $BACKUP_RETENTION_DAYS

User=backup
StandardOutput=journal
StandardError=journal
```

---

## ЧАСТЬ 6: Restart и Recovery политики

### Restart опции

```ini
[Service]
# Никогда не перезапускать (по умолчанию)
Restart=no

# Перезапускать только при ошибке (exit code != 0)
Restart=on-failure

# Перезапускать при abnormal exit (crash, timeout)
Restart=on-abnormal

# Перезапускать при abnormal exit или watchdog timeout
Restart=on-watchdog

# Всегда перезапускать (даже при нормальном выходе)
Restart=always

# Перезапускать только при успехе (exit code 0)
Restart=on-success
```

### RestartSec (задержка перед перезапуском)

```ini
[Service]
Restart=on-failure
RestartSec=5

# Запустится, упадёт
# systemd подождёт 5 секунд
# Запустится снова
```

### StartLimitInterval и StartLimitBurst

```ini
[Service]
Restart=on-failure
RestartSec=5

# Не более 5 перезапусков в течение 60 секунд
StartLimitInterval=60
StartLimitBurst=5

# Если больше - не будет перезапускаться, пока не истечёт интервал
```

### Практический пример (production-ready)

```ini
[Unit]
Description=Production Web App
After=network.target

[Service]
Type=simple
ExecStart=/opt/web-app/run.sh

# Recovery политика
Restart=on-failure
RestartSec=10
StartLimitInterval=300
StartLimitBurst=3

# Если 3 краша за 300 сек (5 мин) - остановиться
# Попробует перезапусться макимум 3 раза за 5 минут

User=www-data
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## ЧАСТЬ 7: Управление service и timer файлами

### Создание и регистрация

```bash
# 1. Создать service файл
sudo nano /etc/systemd/system/my-service.service

# 2. Перезагрузить systemd (прочитать новые файлы)
sudo systemctl daemon-reload

# 3. Включить сервис
sudo systemctl enable my-service.service

# 4. Запустить
sudo systemctl start my-service.service
```

### Управление

```bash
# Посмотреть все сервисы
systemctl list-units --type=service

# Посмотреть все таймеры
systemctl list-timers

# Статус сервиса
sudo systemctl status my-service.service

# Логи
journalctl -u my-service.service
journalctl -u my-service.service -f  # Follow (live)
journalctl -u my-service.service -n 100  # 100 последних строк

# Перезагрузить конфиг
sudo systemctl daemon-reload

# Перезапустить
sudo systemctl restart my-service.service

# Остановить
sudo systemctl stop my-service.service

# Отключить (не запускаться при загрузке)
sudo systemctl disable my-service.service

# Удалить сервис
sudo rm /etc/systemd/system/my-service.service
sudo systemctl daemon-reload
```

### Редактирование после создания

```bash
# Редактировать service файл
sudo systemctl edit my-service.service
# Откроется редактор, изменения сохранятся

# Редактировать через nano/vim
sudo nano /etc/systemd/system/my-service.service
# Потом не забыть:
sudo systemctl daemon-reload

# Посмотреть что изменилось
sudo systemctl show my-service.service
```

---

## ЧАСТЬ 8: Практический сценарий - Backup с таймером

### Создание backup сервиса и таймера

**Шаг 1:** Создать backup скрипт
```bash
sudo nano /usr/local/bin/backup.sh
```

```bash
#!/bin/bash

# Backup скрипт с 3-2-1 правилом

BACKUP_SOURCE="/home"
BACKUP_DEST="/mnt/backup"
DAILY_DEST="$BACKUP_DEST/daily"
ARCHIVE="/tmp/backup-$(date +%Y%m%d-%H%M%S).tar.gz"

# Создать backup
tar -czf "$ARCHIVE" "$BACKUP_SOURCE"

# Скопировать на локальный диск
cp "$ARCHIVE" "$DAILY_DEST/"

# Скопировать на USB (если смонтирован)
if [ -d "/mnt/usb/backup" ]; then
  cp "$ARCHIVE" "/mnt/usb/backup/"
fi

# Скопировать на облако (если есть)
if command -v rclone &> /dev/null; then
  rclone copy "$ARCHIVE" "remote:backup/"
fi

# Удалить старые backup'ы (старше 30 дней)
find "$DAILY_DEST" -name "*.tar.gz" -mtime +30 -delete

# Логирование
logger "Backup completed: $ARCHIVE"
```

```bash
sudo chmod +x /usr/local/bin/backup.sh
```

**Шаг 2:** Создать service файл
```bash
sudo nano /etc/systemd/system/backup.service
```

```ini
[Unit]
Description=Daily Backup Service
After=network.target
Documentation=https://example.com/backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
Group=backup

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Шаг 3:** Создать timer файл
```bash
sudo nano /etc/systemd/system/backup.timer
```

```ini
[Unit]
Description=Daily Backup Timer
Documentation=https://example.com/backup

[Timer]
# Запускать в 2:00 AM каждый день
OnCalendar=*-*-* 02:00:00
# Если систему загрузили между 2:00 и 3:00 - все равно выполнить
Persistent=true

Unit=backup.service

[Install]
WantedBy=timers.target
```

**Шаг 4:** Включить и запустить

```bash
# Перезагрузить systemd
sudo systemctl daemon-reload

# Включить таймер
sudo systemctl enable backup.timer

# Запустить таймер
sudo systemctl start backup.timer

# Проверить статус
sudo systemctl list-timers backup.timer

# Протестировать (запустить сейчас)
sudo systemctl start backup.service

# Посмотреть логи
journalctl -u backup.service -f
```

---

## ЧАСТЬ 9: Шпаргалка (быстрая справка)

### Основные команды

```bash
# Просмотр
systemctl list-units --type=service
systemctl list-timers
sudo systemctl status my-service.service
journalctl -u my-service.service -f

# Управление
sudo systemctl daemon-reload       # После изменений в файле
sudo systemctl enable my-service   # Включить при загрузке
sudo systemctl start my-service    # Запустить сейчас
sudo systemctl stop my-service     # Остановить
sudo systemctl restart my-service  # Перезапустить
sudo systemctl disable my-service  # Отключить при загрузке

# Редактирование
sudo systemctl edit my-service     # Безопасное редактирование
sudo nano /etc/systemd/system/my-service.service  # Через редактор
```

### Структура файлов

```bash
# Service файл:
/etc/systemd/system/my-service.service
[Unit] - описание и зависимости
[Service] - как запускать
[Install] - как интегрировать

# Timer файл:
/etc/systemd/system/my-timer.timer
[Unit] - описание
[Timer] - расписание
[Install] - включение
```

### Типичные ошибки

```bash
# ❌ Забыл daemon-reload
sudo systemctl start my-service
# Unit my-service.service not found

# ✅ Правильно:
sudo systemctl daemon-reload
sudo systemctl start my-service

# ❌ Неправильные права на скрипт
[Service]
ExecStart=/usr/local/bin/script.sh
# Permission denied

# ✅ Правильно:
chmod +x /usr/local/bin/script.sh

# ❌ Нет логирования
[Service]
ExecStart=/opt/app/run.sh
# Нет видно логов!

# ✅ Правильно:
StandardOutput=journal
StandardError=journal
```

---

## Связанные заметки

### ← Перед этим (обязательно)
- [[systemd-basics]] - основы systemd (сервисы, journalctl, состояние)

### → Практическое применение
- [[arch-maintenance-guide]] - использование в обслуживании системы
- [[linux-backup-strategy]] - таймеры для автоматизации backup'ов

### ↔ Параллельно
- [[linux-users-groups]] - системные пользователи для сервисов

### 📚 Главный индекс
- [[00-start-here-index]] - полная навигация по базе знаний

---

## Источники

- `man systemd.service` - полная документация service файлов
- `man systemd.timer` - полная документация таймеров
- `man systemd.unit` - общие свойства Unit файлов
- `man systemctl` - управление сервисами
- `man journalctl` - просмотр логов
- Freedesktop.org: systemd documentation
- Arch Wiki: systemd

---

Создано: 2026-01-04