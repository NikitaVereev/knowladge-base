---
title: 4. Отладка (Debugging)
type: tutorial
tags: [docker, tutorial, debug, logs, exec, network, troubleshooting]
---

# 4: Лабораторная работа по отладке

В реальной жизни контейнеры падают. В этом уроке мы специально "сломаем" контейнеры и научимся их чинить, используя инструменты диагностики.

## Сценарий 1: Контейнер падает сразу после старта

Попробуем запустить "сломанный" образ.

1.  Запустите:
    ```bash
    docker run -d --name broken-app alpine:3.18 sh -c "echo Starting... && sleep 1 && exit 1"
    ```

2.  Проверьте статус:
    ```bash
    docker ps -a
    # STATUS: Exited (1) ...
    ```

3.  **Диагностика:**
    *   Смотрим логи: `docker logs broken-app`. Видим "Starting...". Но почему упал?
    *   Смотрим детали:
        ```bash
        docker inspect broken-app --format='{{.State.ExitCode}}'
        ```
        Код `1` означает ошибку приложения.

4.  **Отладка:**
    Запустим контейнер в интерактивном режиме, перезаписав команду, чтобы "зайти и осмотреться".
    ```bash
    docker run -it --rm --entrypoint sh alpine:3.18
    # Теперь мы внутри. Можно пробовать запускать команды вручную.
    ```

## Сценарий 2: Нет связи с базой данных (Сетевая магия)

Создадим ситуацию, когда приложение не видит БД.

1.  Создайте сеть:
    ```bash
    docker network create my-net
    ```

2.  Запустите Redis в этой сети:
    ```bash
    docker run -d --name my-redis --network my-net redis
    ```

3.  Запустите "приложение" (Alpine) **В ДРУГОЙ** сети (дефолтной):
    ```bash
    docker run -it --rm --name my-client alpine
    ```

4.  Попробуйте пинговать Redis изнутри `my-client`:
    ```bash
    ping my-redis
    # ping: bad address 'my-redis'
    ```
    *Проблема: Контейнеры в разных сетях изолированы.*

5.  **Решение (на лету):**
    Не выходя из контейнера `my-client` (откройте новый терминал хоста):
    ```bash
    # Подключаем работающий контейнер к сети
    docker network connect my-net my-client
    ```

6.  Вернитесь в терминал `my-client` и попробуйте снова:
    ```bash
    ping my-redis
    # PING my-redis (172.18.0.2): 56 data bytes... (УСПЕХ!)
    ```

## Сценарий 3: "Где мои файлы?" (Volume Inspect)

Вы запустили базу данных, но не знаете, где физически лежат файлы на вашем Mac/Linux.

1.  Запустите Postgres с томом:
    ```bash
    docker run -d --name pg-db -v pg-data:/var/lib/postgresql/data postgres:15-alpine
    ```

2.  Найдите точку монтирования:
    ```bash
    docker volume inspect pg-data
    ```
    *   **Linux:** Вы увидите путь `/var/lib/docker/volumes/pg-data/_data`.
    *   **Mac/Windows:** Вы увидите путь *внутри виртуальной машины Docker*. Вы не можете открыть его в Finder/Explorer напрямую.

3.  **Хак для просмотра данных (работает везде):**
    Используйте временный контейнер:
    ```bash
    docker run --rm -v pg-data:/data alpine ls -la /data
    ```
    Вы увидите файлы базы (`base`, `global`, `pg_hba.conf`...).

## Итоги
1.  Если контейнер падает (`Exited`), смотрите `docker logs` и `ExitCode`.
2.  Если нет связи (`ping` fails), проверьте `docker network ls` и подключение контейнеров.
3.  Если нужно проверить данные в Volume, используйте `docker run --rm -v ... alpine ls`.
