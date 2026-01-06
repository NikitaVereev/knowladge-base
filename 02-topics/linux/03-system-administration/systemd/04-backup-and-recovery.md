---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# 04: Резервное копирование и восстановление

## 🎯 ТРИ ОСНОВНЫХ ПОДХОДА

### 1. **rsync** — Традиционный (надежный)
- Копирование файлов
- Инкрементальный
- SSH поддержка
- Простой

### 2. **restic** — Современный (шифрование)
- Полное шифрование
- Облачные backends
- Incremental snapshots
- Верификация данных

### 3. **rclone** — Облако-ориентированный
- Любое облако (S3, B2, Google Drive)
- Синхронизация
- Шифрование

**Выбор зависит от вам:**
- Локальный backup + SSH? → rsync
- Облако + шифрование? → restic
- Облако любое? → rclone

---

## 📦 rsync: КОПИРОВАНИЕ ФАЙЛОВ

### Базовые команды

```bash
# Базовая копия
rsync -av /source/ /destination/

# С удалением (синхронизация)
rsync -av --delete /source/ /destination/

# Через SSH
rsync -avz user@remote:/remote/path/ /local/path/

# С исключением файлов
rsync -av --exclude="*.log" --exclude="*.tmp" /source/ /dest/

# Драй-ран (без изменений)
rsync -av --dry-run /source/ /destination/
```

### Пример: Backup /home

```bash
#!/bin/bash
SOURCE="/home/"
DEST="/mnt/backup/home-$(date +%Y%m%d)"

rsync -av \
  --delete \
  --exclude="Cache" \
  --exclude=".mozilla/firefox/*/cache*" \
  $SOURCE $DEST

echo "Backup complete: $DEST"
```

### Cron job

```bash
# Редактировать crontab
crontab -e

# Добавить (каждый день в 2 AM)
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

---

## 💾 restic: СОВРЕМЕННЫЙ BACKUP (ШИФРОВАНИЕ)

### Инициализация

```bash
# Локальный backup
mkdir /mnt/backup/restic
restic init -r /mnt/backup/restic

# S3 backup (AWS)
export AWS_ACCESS_KEY_ID="xxx"
export AWS_SECRET_ACCESS_KEY="yyy"
restic init -r s3:s3.amazonaws.com/mybucket/restic

# Backblaze B2
export B2_ACCOUNT_ID="xxx"
export B2_ACCOUNT_KEY="yyy"
restic init -r b2:mybucket:/restic
```

### Backup команды

```bash
# Backup директория
restic -r /mnt/backup/restic backup /home/user/documents

# Backup с исключением
restic -r /mnt/backup/restic backup \
  --exclude="*.tmp" \
  --exclude="Cache" \
  /home/user

# Множество директорий
restic -r /mnt/backup/restic backup /home /etc /var
```

### Restore команды

```bash
# Список snapshots
restic -r /mnt/backup/restic snapshots

# Restore последний
restic -r /mnt/backup/restic restore latest --target /tmp/restore

# Restore конкретный файл
restic -r /mnt/backup/restic dump latest /home/user/important.txt > /tmp/important.txt
```

### Проверка и обслуживание

```bash
# ✅ ВАЖНО: Проверить целостность
restic -r /mnt/backup/restic check

# Показать disk usage
restic -r /mnt/backup/restic stats

# Удалить старые snapshots (старше 90 дней)
restic -r /repo forget --keep-daily 30 --keep-monthly 12 --prune
```

---

## 🌐 rclone: ОБЛАЧНОЕ КОПИРОВАНИЕ

### Команды

```bash
# Копировать локально → облако
rclone copy /home/user/photos mycloudname:backups/photos

# Синхронизация (с удалением на облаке)
rclone sync /home/user/docs mycloudname:backups/docs

# С шифрованием
rclone copy /home/user mycloudname:encrypted --crypt-filename-encryption standard

# Список файлов
rclone ls mycloudname:backups/
```

---

## 🎯 3-2-1 RULE (для важных данных)

```
3 копии total (+ 2 backups)
2 разные носители
1 offsite копия
```

**Реализация:**
```bash
# Copy 1: Оригинал на диске
/home/user/important

# Copy 2: Локальный backup (rsync)
/mnt/backup/important

# Copy 3: Облачный backup (restic)
restic backup /home/user/important → S3
```

---

## 📋 ШПАРГАЛКА BACKUP

```bash
# rsync
rsync -av /source/ /destination/        # Copy
rsync -av --delete /source/ /dest/      # Sync

# restic
restic init -r /repo                    # Initialize
restic -r /repo backup /home            # Backup
restic -r /repo snapshots               # List
restic -r /repo restore latest /tmp     # Restore
restic -r /repo check                   # Verify

# rclone
rclone copy local remote                # Copy
rclone sync local remote                # Sync
```

---

## 🔗 ДАЛЬШЕ

→ [05-system-monitoring.md](./05-system-monitoring.md)
