---
title: 4 Работа с Volumes - Бэкап и Восстановление
description: Сценарии Disaster Recovery - создание "горячих" дампов баз данных, "холодный" бэкап файлов тома через tar и миграция данных между хостами.
---

## 1. Бэкап Баз Данных (Logical Backup)
Самый надежный способ — использовать нативные утилиты БД (`pg_dump`, `mysqldump`) внутри контейнера. Не нужно останавливать базу.

### PostgreSQL
```bash
# Дамп потоком сразу в файл на хосте
docker exec -t my-postgres pg_dumpall -c -U postgres > dump_$(date +%F).sql
```

### MySQL / MariaDB
```bash
docker exec my-mysql mysqldump -u root --password=secret --all-databases > dump.sql
```

**Восстановление (Restore):**
```bash[[4-backup-and-restore]]
# Входной поток из файла на хосте внутрь контейнера
cat dump.sql | docker exec -i my-postgres psql -U postgres
```

## 2. Бэкап Файлов Тома (Filesystem Backup)
Если у вас не БД, а просто файлы (uploads, jenkins_home), или вам нужен бинарный бэкап.

**Техника "Busybox-Helper"**:
Мы запускаем крошечный временный контейнер, монтируем к нему том, архивируем его и удаляемся.

### Создание архива (Backup)
Предположим, том называется `my-data`.
```bash
docker run --rm \
  -v my-data:/data \            # Монтируем том в /data
  -v $(pwd):/backup \           # Монтируем текущую папку хоста в /backup
  alpine \
  tar czf /backup/backup.tar.gz -C /data .
```
*Результат*: Файл `backup.tar.gz` появится в текущей папке.

### Развертывание архива (Restore)
Восстанавливаем данные в **новый** или очищенный том `my-data-restored`.
```bash
docker run --rm \
  -v my-data-restored:/data \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /data && tar xzf /backup/backup.tar.gz"
```

## 3. Миграция данных (Clone Volume)
Как скопировать данные из тома `vol-old` в том `vol-new`?

```bash
docker run --rm \
  -v vol-old:/from \
  -v vol-new:/to \
  alpine \
  ash -c "cd /from && cp -av . /to"
```
Флаг `-a` (archive) важен, чтобы сохранить права доступа и владельцев файлов.

## 4. Обслуживание

### Список томов
```bash
docker volume ls
```

### Удаление неиспользуемых (Orphaned)
Тома не удаляются автоматически при удалении контейнера (если не указать `-v`). Со временем они накапливаются.
```bash
docker volume prune
```

### Анализ занимаемого места
```bash
docker system df -v
```
Эта команда покажет, какие тома сколько места занимают и используются ли они.
