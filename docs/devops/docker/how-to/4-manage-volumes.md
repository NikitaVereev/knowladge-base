---
title: "4 Работа с Volumes (Бэкапы)"
description: "Руководство по управлению томами, созданию инкрементальных бэкапов и восстановлению данных в Docker."
---

Docker Volumes — это предпочтительный механизм для хранения постоянных данных (базы данных, файлы пользователей), так как они не зависят от жизненного цикла контейнера и файловой системы хоста.

## Основные команды управления

*   **Создать том:** `docker volume create pg_data`
*   **Список томов:** `docker volume ls`
*   **Удалить том:** `docker volume rm pg_data`
    > *Важно:* Вы не сможете удалить том, если он смонтирован в какой-либо контейнер (даже остановленный). Сначала удалите контейнер (`docker rm`).
*   **Очистка (Prune):** `docker volume prune` — удалит все тома, не используемые контейнерами. *Будьте осторожны, это необратимо!*
*   **Инспекция:** `docker volume inspect pg_data` — покажет точку монтирования на хосте (Mountpoint).

## Резервное копирование (Backup)

Docker не имеет встроенной команды `docker backup`. Стандартный подход (Best Practice) — использование временного служебного контейнера.

### Шаг 1: Остановка контейнера (Рекомендуется)
Для баз данных (Postgres/MySQL) копирование файлов на лету может привести к битым данным. Лучше остановить контейнер.
```bash
docker stop my-db
```

### Шаг 2: Создание архива
Мы запускаем маленький контейнер (alpine), монтируем в него наш volume и папку для бэкапа, и архивируем данные.

```bash
docker run --rm \
  -v pg_data:/volume-data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pg_data_backup.tar.gz -C /volume-data .
```
*   `--rm`: Удалить контейнер сразу после выполнения.
*   `-v pg_data:/volume-data`: Подключаем наш том внутрь контейнера.
*   `-v $(pwd):/backup`: Подключаем текущую папку хоста.
*   `tar czf ...`: Команда внутри контейнера, которая создает архив.

### Шаг 3: Запуск обратно
```bash
docker start my-db
```

> **Совет:** Для баз данных надежнее использовать их родные утилиты (`pg_dump`, `mysqldump`) через `docker exec`, чем копировать файлы тома.
> ```bash
> docker exec -t my-db pg_dumpall -c -U postgres > dump.sql
> ```

## Восстановление (Restore)

Процедура обратная бэкапу.

1.  **Создаем новый пустой том:**
    ```bash
    docker volume create pg_data_restored
    ```

2.  **Распаковываем архив во временном контейнере:**
    ```bash
    docker run --rm \
      -v pg_data_restored:/volume-data \
      -v $(pwd):/backup \
      alpine sh -c "cd /volume-data && tar xzf /backup/pg_data_backup.tar.gz"
    ```

3.  **Запускаем контейнер с новым томом:**
    ```bash
    docker run -d --name new-db -v pg_data_restored:/var/lib/postgresql/data postgres:16
    ```
