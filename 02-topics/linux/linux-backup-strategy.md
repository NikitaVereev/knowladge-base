---
created: 2026-01-03
tags: [linux, backup, data-protection, system-administration, reference]
type: reference
---

# Linux Backup Strategy - стратегия резервного копирования данных

## Основная идея

Backup - это не о том когда нужно восстанавливать, это о том чтобы НИКОГДА не нужно было восстанавливать через восстановление.

**Три закона backup'а (3-2-1 rule):**
- **3 копии** данных (original + 2 backup)
- **2 разные носители** (диск + облако, USB + NAS)
- **1 копия off-site** (в другом месте, не дома)

**Что можно потерять:**
- Диск выходит из строя (отказ железа)
- Вирус шифрует данные (ransomware)
- Случайное удаление
- Пожар / наводнение / кража
- Взлом учетной записи

---

## ЧАСТЬ 1: Типы backup'ов

### Full backup (полный)

```bash
# ОПИСАНИЕ:
# - Копирует ВСЕ файлы
# - Размер большой (может быть ТБ)
# - Восстановление быстрое (всё на месте)
# - Время создания долгое

# КОГДА:
# - Первый раз
# - Еженедельно

# ПРИМЕР:
tar -czf backup-full-$(date +%Y-%m-%d).tar.gz /home/user/documents

# Размер:
# /home/user/documents = 50GB
# backup-full-2026-01-03.tar.gz = 45GB (сжато)
```

### Incremental backup (добавочный)

```bash
# ОПИСАНИЕ:
# - Копирует ТОЛЬКО файлы изменённые с предыдущего backup'а
# - Размер маленький (может быть МБ)
# - Восстановление медленнее (нужно развернуть full + все incremental)
# - Время создания быстрое

# КОГДА:
# - Ежедневно
# - Несколько раз в день

# ПРИМЕР:
tar -czf backup-incr-$(date +%Y-%m-%d).tar.gz \
    --newer-mtime-than backup-full-2026-01-01.tar.gz \
    /home/user/documents

# Размер:
# День 1: 2.5GB
# День 2: 500MB
# День 3: 300MB
# Итого за неделю: full (45GB) + 5*incremental (3GB) = 48GB
```

### Differential backup (дельта)

```bash
# ОПИСАНИЕ:
# - Копирует файлы изменённые с ПОЛНОГО backup'а
# - Размер растёт со временем (но не как incremental)
# - Восстановление быстрее чем incremental (только full + last differential)
# - Компромисс между full и incremental

# КОГДА:
# - Несколько раз в неделю

# ПРИМЕР:
tar -czf backup-diff-$(date +%Y-%m-%d).tar.gz \
    --newer-mtime-than backup-full-2026-01-01.tar.gz \
    /home/user/documents

# Размер:
# День 1: full (45GB)
# День 2: diff (1.5GB)  - изменённое за 1 день
# День 3: diff (2.5GB)  - изменённое за 2 дня
# День 4: diff (3.5GB)  - изменённое за 3 дня
```

### Таблица сравнения

| Тип | Размер | Время | Восстановление | Когда |
|-----|--------|-------|-----------------|-------|
| Full | Большой | Долго | Быстро (direct) | Еженедельно |
| Incremental | Маленький | Быстро | Медленно (chain) | Ежедневно |
| Differential | Средний | Быстро | Среднее (full+last) | 2-3x в неделю |

---

## ЧАСТЬ 2: Стратегия 3-2-1

### Стратегия для домашнего пользователя

```
ORIGINAL (Production)
├── /home/user/documents (на SSD)
└── /home/user/projects (на SSD)

BACKUP #1 (Local, Hot)
├── /mnt/backup-disk/documents (External HDD, connected)
└── Обновляется: ежедневно

BACKUP #2 (Local, Cold)
├── /mnt/nas/documents (NAS в сети)
└── Обновляется: еженедельно

BACKUP #3 (Off-site)
├── Облако: Google Drive / Yandex.Disk
└── Обновляется: ежемесячно
```

### Стратегия для production сервера

```
ORIGINAL (Production)
├── /var/www/app (на RAID-1 диск)
└── /var/lib/postgresql (на RAID-1 диск)

BACKUP #1 (Local, Hot)
├── Снимок filesystem (btrfs / LVM snapshot)
└── Обновляется: ежедневно (или раз в час!)

BACKUP #2 (Local, Cold)
├── На отдельном физическом диске
└── Обновляется: еженедельно

BACKUP #3 (Off-site)
├── S3 (AWS) или Google Cloud Storage
└── Обновляется: ежедневно
```

### Схема ротации backup'ов (GFS - Grandfather Father Son)

```
WEEKLY (Полный)
├── Full backup каждый понедельник
└── Хранить 4 недели (1 месяц)

DAILY (Добавочный)
├── Incremental backup каждый день (Вт-Вс)
└── Хранить 7 дней (перезаписываются)

MONTHLY (Full)
├── Full backup в конце месяца
└── Хранить 12 месяцев (1 год)

YEARLY (Full)
├── Full backup в конце года
└── Хранить 7 лет (для архива)
```

**Пример реализации:**
```bash
# Понедельник (Weekly full)
tar -czf /backup/weekly/week-$(date +%W).tar.gz /home/user/

# Вторник-Воскресенье (Daily incremental)
tar -czf /backup/daily/day-$(date +%u).tar.gz \
    --newer-mtime-than /backup/weekly/week-$(date +%W).tar.gz \
    /home/user/

# Конец месяца (Monthly full)
tar -czf /backup/monthly/$(date +%Y-%m).tar.gz /home/user/

# Конец года (Yearly full)
tar -czf /backup/yearly/$(date +%Y).tar.gz /home/user/
```

---

## ЧАСТЬ 3: Инструменты backup'а

### tar (самый простой)

```bash
# Базовая операция
tar -czf backup.tar.gz /path/to/backup

# Параметры:
# -c = create (создать архив)
# -z = gzip (сжать)
# -f = file (файл)
# -v = verbose (вывод прогресса)

# Примеры:

# 1. Полный backup documents
tar -czf ~/backup-documents-$(date +%Y-%m-%d).tar.gz \
    /home/user/documents

# 2. Только изменённые файлы за день
tar -czf ~/backup-today.tar.gz \
    --newer-mtime-than /tmp/last-backup-time \
    /home/user/

# 3. Исключить некоторые папки
tar -czf ~/backup.tar.gz \
    --exclude='.cache' \
    --exclude='node_modules' \
    /home/user/

# 4. С прогрессом
tar -czvf ~/backup.tar.gz /home/user/

# ВОССТАНОВЛЕНИЕ:

# Распаковать всё
tar -xzf backup.tar.gz

# Распаковать в конкретную папку
tar -xzf backup.tar.gz -C /restore/location

# Посмотреть что внутри
tar -tzf backup.tar.gz | head
```

### rsync (для синхронизации)

```bash
# ОПИСАНИЕ:
# - Синхронизирует две папки
# - Передаёт только ИЗМЕНЁННЫЕ части файлов
# - Очень эффективно для сетевых backup'ов

# Базовая операция:
rsync -av /source /destination

# Параметры:
# -a = archive (сохраняет права, временные метки)
# -v = verbose (вывод)
# -z = compress (сжимает при передаче по сети)
# --delete = удаляет файлы в destination которых нет в source

# ПРИМЕРЫ:

# 1. Local backup (на external диск)
rsync -av /home/user/documents /mnt/backup-disk/

# 2. На NAS (по сети)
rsync -avz /home/user/documents rsync://nas.local/backup/

# 3. На сервер (SSH)
rsync -avz -e ssh /home/user/documents user@server.com:/backup/

# 4. С удалением отсутствующих файлов
rsync -avz --delete /home/user/documents /mnt/backup/

# 5. С исключениями
rsync -avz --exclude='.git' --exclude='node_modules' \
    /home/user/projects /mnt/backup/

# 6. Dry-run (посмотреть что будет скопировано без копирования)
rsync -avz --dry-run /home/user/ /mnt/backup/

# ПРЕИМУЩЕСТВА vs tar:
# - Incremental по-умолчанию (только изменённое)
# - Сохраняет структуру (не архивирует)
# - Удаляет старые файлы (--delete)
# - Работает по сети (SSH, rsync protocol)
```

### restic (современный backup tool)

```bash
# УСТАНОВКА:
sudo pacman -S restic

# ИНИЦИАЛИЗАЦИЯ:
# Создать backup репозиторий
restic init --repo /mnt/backup-disk/repo

# Вывод:
# repository 1a2b3c4d opened successfully
# created restic backup repository at /mnt/backup-disk/repo

# СОЗДАНИЕ BACKUP:
restic -r /mnt/backup-disk/repo backup /home/user/documents

# Вывод:
# snapshot 1a2b3c4d created

# СПИСОК BACKUP'ОВ:
restic -r /mnt/backup-disk/repo snapshots

# Вывод:
# ID        | Time                 | Hostname | Size
# 1a2b3c4d  | 2026-01-01 10:00:00 | myhost   | 45 GB
# 2b3c4d5e  | 2026-01-02 10:00:00 | myhost   | 45.5 GB

# ВОССТАНОВЛЕНИЕ:
restic -r /mnt/backup-disk/repo restore 1a2b3c4d --target /restore/

# УДАЛЕНИЕ СТАРЫХ BACKUP'ОВ (retention policy):
restic -r /mnt/backup-disk/repo forget --keep-daily 7 --keep-monthly 12

# ПРОВЕРКА ЦЕЛОСТНОСТИ:
restic -r /mnt/backup-disk/repo check

# ПРЕИМУЩЕСТВА restic:
# - Дедупликация (одинаковые файлы хранятся один раз)
# - Шифрование (по-умолчанию)
# - Инкрементальные backup'ы (по-умолчанию)
# - Versioning (сохраняет старые версии файлов)
# - Быстрое восстановление (любой файл можно восстановить)
```

### Backup БД (PostgreSQL, MySQL)

```bash
# PostgreSQL:

# Full backup целой БД
pg_dump -U username database_name > backup.sql

# С сжатием
pg_dump -U username database_name | gzip > backup.sql.gz

# Все БД
pg_dumpall -U username > all-databases.sql

# Binary backup (WAL архив)
# Требует конфигурации но даёт point-in-time recovery

# ВОССТАНОВЛЕНИЕ:
psql -U username database_name < backup.sql

# MySQL:

# Full backup целой БД
mysqldump -u username -p database_name > backup.sql

# Все БД
mysqldump -u username -p --all-databases > all-databases.sql

# С блокировкой (consistency)
mysqldump -u username -p --single-transaction database_name > backup.sql

# ВОССТАНОВЛЕНИЕ:
mysql -u username -p database_name < backup.sql
```

---

## ЧАСТЬ 4: Практические скрипты

### Script: ежедневный backup (tar + rsync)

```bash
#!/bin/bash
# Сохранить как: /usr/local/bin/daily-backup.sh

set -e

# Конфигурация
SOURCE="/home/user/documents"
BACKUP_DISK="/mnt/backup-disk"
LOG_FILE="/var/log/backup.log"

# Функция логирования
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$LOG_FILE"
}

log "=== Starting daily backup ==="

# Проверить что backup диск mounted
if ! mountpoint -q "$BACKUP_DISK"; then
    log "ERROR: Backup disk not mounted!"
    exit 1
fi

# Проверить место на диске
AVAILABLE=$(df "$BACKUP_DISK" | awk 'NR==2 {print $4}')
if [ "$AVAILABLE" -lt 5242880 ]; then  # 5GB в KB
    log "ERROR: Not enough space on backup disk! Available: ${AVAILABLE}KB"
    exit 1
fi

# Создать backup
BACKUP_FILE="$BACKUP_DISK/daily-$(date +%Y-%m-%d).tar.gz"

log "Starting tar backup to $BACKUP_FILE"
if tar -czf "$BACKUP_FILE" "$SOURCE" 2>> "$LOG_FILE"; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log "Backup successful! Size: $SIZE"
else
    log "ERROR: Backup failed!"
    exit 1
fi

# Очистить старые backup'ы (старше 7 дней)
log "Cleaning old backups..."
find "$BACKUP_DISK" -name "daily-*.tar.gz" -mtime +7 -delete

log "=== Backup completed successfully ==="
```

**Использование:**
```bash
# Сделать исполняемым
chmod +x /usr/local/bin/daily-backup.sh

# Запустить вручную
/usr/local/bin/daily-backup.sh

# Или через cron (каждый день в 2:00)
# Добавить в crontab -e:
0 2 * * * /usr/local/bin/daily-backup.sh

# Проверить логи
tail -f /var/log/backup.log
```

### Script: restic backup с проверкой

```bash
#!/bin/bash
# Сохранить как: /usr/local/bin/restic-backup.sh

set -e

RESTIC_REPO="/mnt/backup-disk/restic-repo"
BACKUP_SOURCES="/home/user/documents /home/user/projects"
LOG_FILE="/var/log/restic-backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Starting restic backup ==="

# Проверить репозиторий
if [ ! -d "$RESTIC_REPO" ]; then
    log "Initializing restic repository..."
    restic init --repo "$RESTIC_REPO"
fi

# Создать backup
log "Creating backup..."
restic -r "$RESTIC_REPO" backup $BACKUP_SOURCES

# Показать статус
log "Backup statistics:"
restic -r "$RESTIC_REPO" stats

# Забыть старые backup'ы (retention policy)
log "Applying retention policy..."
restic -r "$RESTIC_REPO" forget --keep-daily 7 --keep-monthly 12 --prune

# Проверить целостность
log "Checking repository integrity..."
restic -r "$RESTIC_REPO" check

log "=== Restic backup completed ==="
```

### Script: backup с уведомлением

```bash
#!/bin/bash
# С отправкой email если backup упал

BACKUP_SCRIPT="/usr/local/bin/daily-backup.sh"
EMAIL="user@example.com"
HOSTNAME=$(hostname)

if ! "$BACKUP_SCRIPT" >> /tmp/backup.log 2>&1; then
    # Ошибка - отправить email
    SUBJECT="⚠️ Backup FAILED on $HOSTNAME"
    cat /tmp/backup.log | mail -s "$SUBJECT" "$EMAIL"
    exit 1
else
    # Успех
    SIZE=$(du -sh /mnt/backup-disk/daily-*.tar.gz | tail -1 | cut -f1)
    SUBJECT="✅ Backup successful on $HOSTNAME (Size: $SIZE)"
    echo "Backup completed successfully" | mail -s "$SUBJECT" "$EMAIL"
fi
```

---

## ЧАСТЬ 5: Облачные хранилища

### Google Drive (rclone)

```bash
# УСТАНОВКА:
sudo pacman -S rclone

# КОНФИГУРАЦИЯ:
rclone config

# Выбрать Google Drive и авторизоваться
# Будет открыт браузер для подтверждения

# ПОСЛЕ КОНФИГУРАЦИИ:

# Посмотреть список конфигов
rclone listremotes

# Список файлов на Google Drive
rclone ls gdrive:/

# Загрузить backup
rclone copy /mnt/backup-disk/backup.tar.gz gdrive:/Backups/

# Синхронизировать папку
rclone sync /home/user/documents gdrive:/Documents/

# С прогрессом и параллельными потоками
rclone sync -P --transfers 4 /home/user/documents gdrive:/Documents/

# Скачать backup обратно
rclone copy gdrive:/Backups/backup.tar.gz /restore/
```

### Yandex.Disk (rclone)

```bash
# КОНФИГУРАЦИЯ:
rclone config

# Выбрать Yandex и авторизоваться

# ИСПОЛЬЗОВАНИЕ:

# Загрузить
rclone copy /mnt/backup-disk/backup.tar.gz yandex:/Backups/

# Синхронизировать
rclone sync /home/user/documents yandex:/Documents/

# Посмотреть место
rclone about yandex:/
```

### S3 (AWS, DigitalOcean Spaces)

```bash
# КОНФИГУРАЦИЯ:
rclone config

# Выбрать S3 и ввести AWS credentials

# ИСПОЛЬЗОВАНИЕ:

# Загрузить
rclone copy /mnt/backup-disk/backup.tar.gz s3://my-backup-bucket/

# С progress
rclone copy -P /mnt/backup-disk/backup.tar.gz s3://my-backup-bucket/

# Синхронизировать с удалением
rclone sync --delete /home/user/documents s3://my-backup-bucket/documents/
```

### Automated cloud backup (systemd timer)

```ini
# /etc/systemd/system/cloud-backup.timer
[Unit]
Description=Daily Cloud Backup Timer
After=network-online.target

[Timer]
OnCalendar=daily
OnBootSec=10m
Persistent=yes

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/cloud-backup.service
[Unit]
Description=Daily Cloud Backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cloud-backup.sh
StandardOutput=journal
StandardError=journal
```

```bash
# /usr/local/bin/cloud-backup.sh
#!/bin/bash
set -e

# Проверить интернет
if ! ping -c 1 8.8.8.8 > /dev/null 2>&1; then
    echo "No internet connection"
    exit 1
fi

# Создать local backup
/usr/local/bin/daily-backup.sh

# Загрузить в облако
LATEST=$(ls -t /mnt/backup-disk/daily-*.tar.gz | head -1)
rclone copy "$LATEST" gdrive:/Backups/

echo "Cloud backup completed"
```

**Включить:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cloud-backup.timer
sudo systemctl list-timers cloud-backup.timer
```

---

## ЧАСТЬ 6: Восстановление данных

### Восстановить файл из tar backup

```bash
# Посмотреть что внутри
tar -tzf backup.tar.gz | grep "filename"

# Восстановить конкретный файл
tar -xzf backup.tar.gz home/user/documents/file.txt -C /

# Восстановить папку
tar -xzf backup.tar.gz home/user/documents/ -C /

# Восстановить всё
tar -xzf backup.tar.gz -C /

# Восстановить в другую папку
tar -xzf backup.tar.gz -C /restore-location/
```

### Восстановить файл из rsync backup

```bash
# Файлы в /mnt/backup-disk/documents/ - это просто копии
# Можно скопировать обратно

cp -r /mnt/backup-disk/documents/file.txt /home/user/documents/

# Или синхронизировать обратно
rsync -av /mnt/backup-disk/documents/ /home/user/documents/
```

### Восстановить файл из restic backup

```bash
# Посмотреть версии файла
restic -r /path/to/repo find file.txt

# Восстановить последнюю версию
restic -r /path/to/repo restore --include="file.txt" --target /restore/

# Восстановить конкретный snapshot
restic -r /path/to/repo restore snapshot-id --target /restore/

# Посмотреть содержимое файла без восстановления
restic -r /path/to/repo dump snapshot-id path/to/file.txt
```

### Восстановить из облачного хранилища

```bash
# Google Drive
rclone copy gdrive:/Backups/backup.tar.gz /restore/

# Yandex.Disk
rclone copy yandex:/Backups/backup.tar.gz /restore/

# S3
rclone copy s3://bucket/backup.tar.gz /restore/

# Потом распаковать:
tar -xzf /restore/backup.tar.gz
```

---

## ЧАСТЬ 7: Проверка backup'ов

### Регулярно проверять backup'ы (ВАЖНО!)

```bash
# МЕСЯЧНАЯ ПРОВЕРКА:

# 1. Проверить что файлы не повреждены
tar -tzf /mnt/backup-disk/backup.tar.gz > /dev/null

# 2. Попробовать восстановить небольшой файл
mkdir /tmp/test-restore
tar -xzf /mnt/backup-disk/backup.tar.gz -C /tmp/test-restore
ls -la /tmp/test-restore
rm -rf /tmp/test-restore

# 3. Проверить место на резервном диске
df -h /mnt/backup-disk

# 4. Проверить что резервные диски всё ещё подключены
lsblk

# 5. Для restic:
restic -r /path/to/repo check

# ЕЖЕГОДНАЯ ПРОВЕРКА:

# 1. Восстановить весь backup на отдельный диск
tar -xzf /mnt/backup-disk/backup.tar.gz -C /mnt/test-restore/

# 2. Проверить целостность файлов
find /mnt/test-restore -type f -exec md5sum {} \; > /tmp/checksums.txt

# 3. Убедиться что восстановленные файлы корректны
# (Открыть документы, проверить содержимое)

# 4. Очистить тестовую папку
rm -rf /mnt/test-restore
```

### Test restore (полное восстановление)

```bash
# Один раз в год полностью восстановить backup на виртуальную машину

# 1. Создать виртуальную машину с Arch Linux
# 2. Установить minimal систему
# 3. Распаковать backup
tar -xzf /mnt/backup-disk/monthly-2026-01.tar.gz -C /

# 4. Проверить что всё работает
# - Открыть файлы
# - Запустить приложения
# - Проверить БД

# 5. Документировать проблемы если есть

# Это ЕДИНСТВЕННЫЙ способ убедиться что backup действительно работает!
```

---

## ЧАСТЬ 8: Шпаргалка (быстрый справочник)

### Основные команды

```bash
# TAR

tar -czf backup.tar.gz /path/to/backup     # Создать
tar -tzf backup.tar.gz | head               # Посмотреть содержимое
tar -xzf backup.tar.gz -C /restore/        # Восстановить

# RSYNC

rsync -av /source /destination              # Sync
rsync -avz -e ssh user@host:/remote /local  # Сетевой sync
rsync -av --delete /source /dest            # С удалением

# RESTIC

restic init --repo /path                    # Инициализация
restic -r /path backup /data                # Создать backup
restic -r /path snapshots                   # Список backup'ов
restic -r /path restore id --target /dest   # Восстановить
restic -r /path check                       # Проверить

# RCLONE

rclone config                               # Конфигурация
rclone copy /local gdrive:/remote           # Загрузить
rclone sync /local gdrive:/remote           # Синхронизировать
rclone ls gdrive:/                          # Список файлов
```

### 3-2-1 Rule Checklist

```
□ ORIGINAL: Данные на основном диске
□ BACKUP #1: На external USB/HDD (еженедельно)
□ BACKUP #2: На NAS или облако (ежемесячно)
□ BACKUP #3: Off-site облако (ежемесячно)

□ Разные типы носителей (минимум 2)
□ Разные места (1 off-site)
□ Протестировано восстановление (год назад)
□ Лог успешных backup'ов (последний месяц)
```

---

## ЧАСТЬ 9: Интеграция с systemd

### Systemd timer для автоматических backup'ов

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00
Persistent=yes
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup Service
After=local-fs.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal
# Отправить email при ошибке:
OnFailure=backup-alert@%n.service
```

```ini
# /etc/systemd/system/backup-alert@.service
[Unit]
Description=Send Backup Alert
ConditionVirtualization=!container

[Service]
Type=oneshot
ExecStart=/usr/local/bin/send-alert.sh
```

**Включить:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
sudo systemctl list-timers backup.timer
sudo journalctl -u backup.service -f
```

---

## ЧАСТЬ 10: Шпаргалка для разных сценариев

### Домашний пользователь

```bash
# SETUP:
# 1. External HDD подключить
sudo mount /dev/sdX /mnt/backup

# 2. Создать weekly backup (full)
tar -czf /mnt/backup/weekly-$(date +%W).tar.gz /home/user/

# 3. Создать daily backup (incremental)
tar -czf /mnt/backup/daily-$(date +%w).tar.gz \
    --newer-mtime-than /mnt/backup/weekly-*.tar.gz \
    /home/user/

# 4. Загрузить в облако (раз в месяц)
rclone copy /mnt/backup/weekly-*.tar.gz gdrive:/Backups/

# ПРОВЕРКА: один раз в год восстановить на тестовый компьютер
```

### Production сервер

```bash
# SETUP:
# 1. Hourly snapshots (btrfs)
systemctl enable --now snapper.timer

# 2. Daily backup на NAS (rsync)
/usr/local/bin/backup-to-nas.sh (every day 02:00)

# 3. Weekly backup на S3 (restic)
/usr/local/bin/backup-to-s3.sh (every week Sunday 04:00)

# ПРОВЕРКА: еженедельно восстановить файлы из бэкапов
```

### Разработчик (git repositories)

```bash
# GitHub/GitLab ОДНА КОПИЯ ТОЛЬКО!
# Это не backup!

# ПРАВИЛЬНО:

# 1. Локальные файлы на SSD
/home/dev/projects/

# 2. Backup на external disk
rsync -av /home/dev/projects/ /mnt/backup/

# 3. Backup в облаке
restic -r /mnt/cloud-repo backup /home/dev/projects

# 4. ПЛЮС GitHub (дополнительная защита)
git push

# GitHub НЕ замена для backup'а!
# GitHub может быть взломан, удалён, пропажен
```

---

## Связанные заметки

- [[arch-maintenance]] - регулярное обслуживание
- [[arch-troubleshooting]] - решение проблем
- [[linux-security-hardening]] - безопасность системы
- [[systemd-basics]] - systemd таймеры

## Источники

- Arch Wiki: Backup programs
- restic documentation: https://restic.readthedocs.io
- rclone documentation: https://rclone.org
- rsync man page: man rsync
- tar man page: man tar

---

Создано: 2026-01-03 21:50

