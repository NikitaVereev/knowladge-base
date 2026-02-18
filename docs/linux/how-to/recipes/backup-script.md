---
title: "Рецепт: Бэкап и восстановление"
type: how-to
tags: [linux, recipe, backup, rsync, tar, dd, cron, restore, 3-2-1]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/04-backup-and-recovery.md"
related:
  - "[[linux/how-to/monitor-system]]"
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/cheatsheet]]"
  - "[[linux/how-to/schedule-tasks]]"
---

# Рецепт: Бэкап и восстановление

> Правило 3-2-1: 3 копии, 2 типа носителей, 1 вне сайта.
> rsync — инкрементальный бэкап. tar — архивы. dd — полная копия диска.

## Стратегия 3-2-1

```
3 копии данных:
  1. Оригинал (рабочий сервер)
  2. Локальная копия (внешний диск / NAS)
  3. Удалённая копия (другой сервер / облако)

2 типа носителей:
  - SSD/HDD на сервере
  - Внешний диск или облачное хранилище

1 копия вне площадки:
  - Другой дата-центр / облако
```

## rsync — инкрементальный бэкап

```bash
# Локальный бэкап
rsync -avz --delete /data/ /backup/data/
# -a  archive (permissions, timestamps, symlinks)
# -v  verbose
# -z  compress
# --delete  удалять файлы в backup которых нет в source

# На удалённый сервер
rsync -avz -e ssh /data/ user@backup-server:/backup/data/

# С исключениями
rsync -avz --exclude='*.log' --exclude='.cache' /home/user/ /backup/home/

# Dry run (посмотреть что будет)
rsync -avzn --delete /data/ /backup/data/
```

## tar — архивы

```bash
# Создать архив
tar czf backup-$(date +%Y%m%d).tar.gz /etc /home /var/www
# c  create
# z  gzip compression
# f  file

# Создать с исключениями
tar czf backup.tar.gz --exclude='*.log' --exclude='.cache' /home/

# Распаковать
tar xzf backup-20250101.tar.gz
tar xzf backup.tar.gz -C /restore/   # в конкретную директорию

# Посмотреть содержимое
tar tzf backup.tar.gz
```

## dd — полная копия диска

```bash
# Образ диска (ОСТОРОЖНО — перепутаете if/of = потеря данных!)
sudo dd if=/dev/sda of=/backup/disk-image.img bs=4M status=progress

# Восстановить диск из образа
sudo dd if=/backup/disk-image.img of=/dev/sda bs=4M status=progress

# Со сжатием
sudo dd if=/dev/sda bs=4M | gzip > /backup/disk.img.gz
gunzip -c /backup/disk.img.gz | sudo dd of=/dev/sda bs=4M
```

## Скрипт автоматического бэкапа

```bash
#!/bin/bash
# /usr/local/bin/backup.sh

set -euo pipefail

# --- Настройки ---
BACKUP_DIR="/backup"
REMOTE_USER="backup"
REMOTE_HOST="backup-server"
REMOTE_DIR="/remote-backup"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M)
LOG="/var/log/backup.log"

# --- Функции ---
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"; }

# --- Локальный бэкап ---
log "Starting local backup..."

# Конфигурации
tar czf "${BACKUP_DIR}/etc-${DATE}.tar.gz" /etc 2>/dev/null

# Домашние директории
rsync -az --delete /home/ "${BACKUP_DIR}/home/"

# Веб-приложения
rsync -az --delete /var/www/ "${BACKUP_DIR}/www/"

# --- Удалённый бэкап ---
log "Syncing to remote..."
rsync -az --delete "${BACKUP_DIR}/" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"

# --- Очистка старых ---
log "Cleaning old backups..."
find "${BACKUP_DIR}" -name "*.tar.gz" -mtime +${RETENTION_DAYS} -delete

log "Backup complete!"
```

```bash
# Права
chmod 700 /usr/local/bin/backup.sh
```

## Автоматизация (cron)

```bash
# Редактировать crontab
sudo crontab -e

# Ежедневно в 3:00
0 3 * * * /usr/local/bin/backup.sh

# Еженедельно (воскресенье в 2:00)
0 2 * * 0 /usr/local/bin/backup.sh
```

Или через systemd timer:

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
```

## Восстановление

```bash
# Из rsync-бэкапа
rsync -avz /backup/home/ /home/

# Из tar-архива
tar xzf /backup/etc-20250101.tar.gz -C /

# Из dd-образа
sudo dd if=/backup/disk.img of=/dev/sda bs=4M status=progress
```

## Важные файлы для бэкапа

| Что | Путь | Зачем |
|-----|------|-------|
| Конфигурации | `/etc/` | Настройки всех сервисов |
| Домашние директории | `/home/` | Данные пользователей |
| Веб-приложения | `/var/www/` | Сайты |
| Базы данных | `pg_dump`, `mysqldump` | Не файлы, а дампы! |
| Crontab | `/var/spool/cron/` | Задачи по расписанию |
| Список пакетов | `pacman -Qqe` / `dpkg --get-selections` | Быстрое восстановление |
| SSH ключи | `~/.ssh/` | Доступы |

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Бэкапят БД копированием файлов | Использовать `pg_dump` / `mysqldump`. Файлы могут быть inconsistent |
| dd перепутали if и of | **Тройная проверка** перед запуском. `of=/dev/sda` = уничтожение данных |
| Не тестируют восстановление | Раз в месяц: восстановить бэкап на тестовом сервере |
| Бэкап на том же диске | Если диск умрёт — потеряете и данные и бэкап. Используйте 3-2-1 |
| Нет уведомлений | Добавить `|| mail -s "Backup FAILED" admin@...` в скрипт |
