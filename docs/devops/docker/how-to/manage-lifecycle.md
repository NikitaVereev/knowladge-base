---
title: "Управление жизненным циклом контейнера"
description: "Как создавать, запускать, останавливать, перезапускать и удалять Docker контейнеры. Полный справочник по командам управления состоянием."
---


Это руководство описывает команды для управления состояниями контейнера: от создания (`Created`) до удаления (`Removed`).

## Создание и запуск

Самая частая команда — `docker run`, которая объединяет создание и запуск.

### Запуск в фоне (Detach)
Для веб-серверов и баз данных:
```bash
docker run --name my-nginx -d nginx:latest
```
* `-d`: Detach mode (фоновый режим).

### Запуск интерактивно
Для утилит и отладки (вход в shell):
```bash
docker run --name my-ubuntu -it ubuntu bash
```
* `-i`: Interactive (держать STDIN открытым).
* `-t`: TTY (выделить псевдо-терминал).

### Только создание (без запуска)
Если нужно подготовить контейнер, но не стартовать:
```bash
docker create --name my-app nginx:latest
docker start my-app
```

## Остановка и перезапуск

### Штатная остановка (Stop)
Отправляет сигнал `SIGTERM`, давая приложению 10 секунд на завершение (закрытие соединений, сброс буферов).
```bash
docker stop my-nginx
# Или с увеличенным таймаутом (например, 20 секунд)
docker stop --time=20 my-nginx
```

### Принудительная остановка (Kill)
Отправляет `SIGKILL` немедленно. Используйте, если приложение зависло.
```bash
docker kill my-nginx
```

### Перезапуск
```bash
docker restart my-nginx
```

### Пауза
"Замораживает" процессы (cgroups freezer), не выгружая из памяти.
```bash
docker pause my-nginx
docker unpause my-nginx
```

## Удаление

Удалить можно только остановленный контейнер (или использовать `-f`).

```bash
# 1. Остановить
docker stop my-nginx
# 2. Удалить
docker rm my-nginx
```

### Массовая очистка
Удалить **все** остановленные контейнеры разом:
```bash
docker container prune
```

## Просмотр состояния

### Список контейнеров
```bash
# Только запущенные
docker ps

# Все (включая остановленные)
docker ps -a

# Только ID (удобно для скриптов)
docker ps -q
```

### Детальная информация (Inspect)
Получить IP, пути к volumes, переменные окружения в формате JSON:
```bash
docker inspect my-nginx

# Получить конкретное поле (например, статус)
docker inspect --format '{{.State.Status}}' my-nginx
```

## Данные и Persistence (Важное замечание)

Контейнеры **stateful** (сохраняют состояние) пока они существуют. Если вы остановите (`stop`) и запустите (`start`) контейнер, файлы внутри сохранятся.

Но если вы сделаете `rm` (удаление) — все данные внутри контейнера (Writable Layer) будут уничтожены. Для постоянного хранения данных используйте **Volumes**.

## Практический пример: Цикл жизни БД

```bash
# 1. Запуск MongoDB
docker run -d --name mongo-db -p 27017:27017 mongo:latest

# 2. Создание файла внутри (эмуляция данных)
docker exec mongo-db touch /data/db/myfile.txt

# 3. Рестарт (данные на месте)
docker restart mongo-db
docker exec mongo-db ls /data/db/myfile.txt  # Файл существует

# 4. Удаление (данные пропадут!)
docker stop mongo-db
docker rm mongo-db
# Теперь контейнера и файла /data/db/myfile.txt больше нет
```

## Связанные материалы

- [[devops/docker/explanation/architecture|Архитектура и состояния контейнеров]]
- [[devops/docker/how-to/logs-and-debugging|Логи и отладка]]
