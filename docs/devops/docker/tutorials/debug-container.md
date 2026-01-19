---
title: "Практика: Отладка MongoDB в Docker"
description: "Пошаговый туториал по диагностике работающего контейнера: создание файлов, проверка персистентности и анализ логов."
---


В этом упражнении мы запустим базу данных, внесем изменения "на живую", проверим логи и удалим контейнер.

## Шаг 1: Запуск
Запустите MongoDB в фоновом режиме:
```bash
docker run -d --name debug-mongo mongo:latest
```

## Шаг 2: Проверка работы
Убедитесь, что процесс поднялся и потребляет ресурсы:
```bash
docker ps
docker stats debug-mongo --no-stream
```

## Шаг 3: "Взлом" контейнера
Зайдем внутрь и создадим тестовый файл, чтобы симулировать данные.
```bash
docker exec -it debug-mongo bash
```
Внутри контейнера:
```bash
echo "Important Data" > /data/db/secret.txt
ls -l /data/db/secret.txt
exit
```

## Шаг 4: Проверка логов
Посмотрим, что MongoDB писала при старте (обычно там инициализация хранилища).
```bash
docker logs debug-mongo | grep "waiting for connections"
```
*Если вывод есть, значит база готова к работе.*

## Шаг 5: Проверка персистентности (Stop/Start)
Остановим и запустим контейнер, чтобы убедиться, что файл на месте (так как мы не удаляли контейнер, его Writable Layer сохранился).
```bash
docker stop debug-mongo
docker start debug-mongo
docker exec debug-mongo cat /data/db/secret.txt
```
*Вывод должен быть: `Important Data`*

## Шаг 6: Очистка
Удалим контейнер (файл пропадет, так как мы не использовали Volume).
```bash
docker rm -f debug-mongo
```

## Связанные материалы

- [[devops/docker/how-to/logs-and-debugging|Справочник по командам отладки]]
