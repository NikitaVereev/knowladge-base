---
title: "Управление Docker образами"
description: "Команды для загрузки, просмотра, удаления, экспорта и импорта образов Docker."
---


## Основные операции

### Поиск и загрузка
Образы обычно загружаются из Docker Hub.

```bash
# Найти образ в реестре
docker search nginx

# Загрузить образ
docker pull nginx:latest
docker pull ubuntu:22.04
```

### Просмотр локальных образов
```bash
docker image ls
# Или короткий алиас:
docker images
```

*Вывод показывает REPOSITORY, TAG, ID и SIZE (виртуальный размер).*

## Очистка места

Со временем на диске скапливается много неиспользуемых образов.

### Удаление конкретного образа
```bash
docker rmi nginx:latest
# Или по ID
docker rmi a87587163612
```

### Удаление "висящих" образов (Dangling)
Dangling images — это слои, которые больше не имеют имени (отображаются как `<none>`). Они появляются, когда вы пересобираете образ с тем же тегом.

```bash
docker image prune
```

### Полная очистка
Удалить **все** образы, которые не используются запущенными контейнерами:
```bash
docker image prune -a
```
*Будьте осторожны: это удалит кэш сборки, и следующий `docker build` будет долгим.*

## Экспорт и импорт (Offline передача)

Если у вас нет доступа к реестру (air-gapped environment), можно передать образ файлом.

### 1. Сохранение в архив (Save)
На машине-источнике:
```bash
docker save -o my-app.tar my-app:v1
# Можно сразу сжать gzip
docker save my-app:v1 | gzip > my-app.tar.gz
```

### 2. Загрузка из архива (Load)
На целевой машине:
```bash
docker load -i my-app.tar
# Или для сжатого
gunzip -c my-app.tar.gz | docker load
```

## Связанные материалы

- [[2-images-and-layers|Как работают слои]]
- [[devops/docker/tutorials/optimize-images|Оптимизация размера образов]]
