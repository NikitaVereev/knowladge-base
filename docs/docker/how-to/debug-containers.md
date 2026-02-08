---
title: "Отладка контейнеров (Debug)"
type: how-to
tags: [docker, debug, logs, netshoot, exec, inspect, troubleshooting]
sources:
  docs: "https://docs.docker.com/engine/reference/commandline/logs/"
related:
  - "[[docker/explanation/runtime-behavior]]"
  - "[[docker/explanation/networking]]"
  - "[[docker/tutorials/05-debugging-workshop]]"
---

# Методики отладки контейнеров

> **TL;DR:** Контейнер падает? `docker logs` → `docker inspect` → `docker exec`.
> Нет shell? Подключи netshoot. Нужно внутрь образа? Используй `dive`.

Когда контейнер падает или ведет себя странно, нужно уметь быстро диагностировать проблему. В этом гайде собраны техники от базового просмотра логов до продвинутого сетевого анализа с использованием sidecar-контейнеров.

## 1. Анализ "мертвого" контейнера

Контейнер упал (`Exited`). Как понять почему?

### Шаг 1: Логи
Смотрим STDOUT/STDERR.
```bash
docker logs <container_name>
# Если логов слишком много, смотрим последние 100 строк
docker logs --tail 100 <container_name>
```

### Шаг 2: Inspect (Код выхода)
Если логов нет (приложение упало до инициализации логгера), смотрим код выхода.
```bash
docker inspect <container_name> --format='{{.State.ExitCode}}'
```
*   `137`: OOM Killer (нехватка памяти).
*   `1`: Ошибка приложения (смотрите код).
*   `0`: Приложение решило, что работа закончена (возможно, неправильный CMD).

### Шаг 3: Переопределение Entrypoint
Если контейнер падает мгновенно (`CrashLoopBackOff`), вы не успеваете зайти внутрь.
Запустите его с командой `sleep`, чтобы он "повис" и дал вам время осмотреться.

```bash
# Переопределяем команду запуска на бесконечный сон
docker run -d --name debug-app --entrypoint sleep my-broken-image infinity

# Заходим внутрь
docker exec -it debug-app sh
# Теперь пробуем запустить приложение вручную и смотрим ошибки
/app/start.sh
```

## 2. Отладка работающего контейнера

### Подключение (Exec)
```bash
docker exec -it <container_name> sh
# Если sh нет (Alpine/Distroless), пробуйте:
# docker exec -it <container_name> /bin/bash
```

### Просмотр процессов
Не используйте `top` внутри контейнера (его может не быть). Используйте docker:
```bash
docker top <container_name>
```

### Проверка файлов (без входа)
Можно скопировать подозрительный конфиг наружу для анализа:
```bash
docker cp <container_name>:/etc/nginx/nginx.conf ./debug_nginx.conf
```

## 3. Сетевая отладка (Netshoot)

Частая проблема: "Приложение не видит базу данных".
В минимальных образах (Alpine, Distroless) нет утилит `curl`, `ping`, `telnet`, `nslookup`. Устанавливать их (`apt-get install`) долго и грязно.

**Решение: Sidecar Container (Netshoot)**
Запустите швейцарский нож сетевой отладки (`nicolaka/netshoot`), подключив его к сети *целевого* контейнера.

```bash
# Подключаем netshoot к сети контейнера 'my-app'
docker run -it --rm --network container:my-app nicolaka/netshoot
```

Теперь вы находитесь "внутри" сетевого стека `my-app`. Вы видите его IP, его порты и `localhost`.
*   `ping google.com` — есть ли интернет?
*   `nslookup db` — работает ли DNS?
*   `nc -zv db 5432` — открыт ли порт базы?
*   `tcpdump -i eth0` — сниффинг трафика.

## 4. Отладка образов (Dive)

Если образ весит слишком много, используйте утилиту **dive**, чтобы увидеть, какой слой сколько занимает.

```bash
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive <image_name>
```
Она покажет дерево файлов и позволит найти "мусор" (например, забытый кэш `apt` или исходники в финальном слое).

## 5. Восстановление данных (Rescue Volume)

Если контейнер с БД умер и не поднимается, нужно спасти данные из Volume.

```bash
# Монтируем volume к временному контейнеру busybox
docker run --rm -v my-db-data:/data -v $(pwd):/backup busybox \
  tar cvf /backup/backup.tar /data
```
Теперь у вас есть архив `backup.tar` на хосте.

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `docker exec` в остановленный контейнер | `Error: container is not running` | Запустить с `docker run --entrypoint sleep ... infinity`, потом exec |
| Distroless — нет bash/sh | `exec: "sh": not found` | Подключить `netshoot` к namespace контейнера или использовать debug image |
| Не смотрят exit code | «Контейнер просто упал» | `docker inspect --format='{{.State.ExitCode}}'` — 137 = OOM, 1 = ошибка приложения |
| Логи обрезаны | Старые логи недоступны | `docker logs --since 1h` или настроить централизованное логирование |