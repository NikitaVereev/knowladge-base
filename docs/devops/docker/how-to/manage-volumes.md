---
title: "Управление данными и томами"
description: "Как создавать Volumes, использовать Bind Mounts для разработки и делать резервные копии данных контейнеров."
---


## Использование Named Volumes

Лучший выбор для баз данных.

```bash
# Создание тома
docker volume create pg-data

# Запуск контейнера с использованием тома
# Данные из /var/lib/postgresql/data будут сохраняться в томе pg-data
docker run -d \
  --name db \
  -v pg-data:/var/lib/postgresql/data \
  postgres:15
```

Даже если вы удалите контейнер (`docker rm -f db`), том `pg-data` останется. Новый контейнер можно запустить с тем же томом, и данные подхватятся.

## Использование Bind Mounts (Для разработки)

Идеально, чтобы прокинуть исходный код внутрь контейнера без пересборки образа.

```bash
# Монтируем текущую директорию $(pwd) в папку /app внутри контейнера
docker run -d \
  --name dev-server \
  -v $(pwd):/app \
  -p 3000:3000 \
  node:18-alpine npm start
```
*Совет: Используйте абсолютные пути (или `$(pwd)`), относительные пути в `-v` могут вести себя непредсказуемо.*

## Резервное копирование тома (Backup)

Docker не имеет встроенной команды "backup volume", но это легко сделать через временный контейнер.

```bash
# Создаем архив backup.tar.gz из содержимого тома my-data-volume
docker run --rm \
  -v my-data-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup.tar.gz -C /data .
```

## Восстановление (Restore)

```bash
# 1. Создаем пустой том
docker volume create my-restored-data

# 2. Распаковываем архив в этот том
docker run --rm \
  -v my-restored-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup.tar.gz -C /data
```

## Копирование файлов (docker cp)

Для разового копирования файлов (например, вытащить логи или конфиг) без монтирования.

```bash
# Из контейнера на хост
docker cp my-container:/app/logs/error.log ./error.log

# С хоста в контейнер
docker cp ./nginx.conf my-nginx:/etc/nginx/nginx.conf
```

## Очистка мусора

Удаление всех томов, которые не используются ни одним контейнером:
```bash
docker volume prune
```

## Связанные материалы

- [[devops/docker/explanation/storage-types|Типы хранилищ: Volumes vs Bind Mounts]]
