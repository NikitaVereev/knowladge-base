---
title: 2 Операции с контейнерами
description: Полное руководство по управлению контейнерами - от базового запуска до политики рестартов, лимитов ресурсов, продвинутого логирования и инспекции JSON.
---

## 1. Запуск и Конфигурация

### Базовый запуск (Detached)
Стандартный способ запуска сервисов (БД, веб-серверов) в фоне.
```bash
docker run -d --name my-postgres postgres:15
```

### Проброс портов и переменных
```bash
docker run -d \
  -p 8080:80 \          # Хост:Контейнер (iptables NAT)
  -e DB_HOST=db \       # ENV переменная
  --name my-app \
  my-image:latest
```

### Смена Entrypoint (Overriding)
Часто нужно запустить контейнер не с основным приложением, а с shell'ом для отладки, игнорируя то, что написано в Dockerfile.
```bash
# --entrypoint полностью заменяет инструкцию ENTRYPOINT в образе
docker run -it --entrypoint /bin/sh my-app

# Если ENTRYPOINT был ["/app/run.sh"], а вы хотите передать аргументы:
docker run my-app --debug-mode
```

### Временные контейнеры (--rm)
Для "одноразовых" задач (миграции БД, тесты, сборка, npm install). Контейнер удалится сам сразу после завершения процесса.
```bash
docker run --rm -v $(pwd):/app node:20 npm install
```

## 2. Управление ресурсами и Жизненным циклом

### Политики перезапуска (Restart Policies)
Страховка от падений приложения или перезагрузки сервера.
```bash
# --restart always: Поднимать всегда (даже если выключил вручную - не рекомендуется)
# --restart unless-stopped: Поднимать, если не было явного docker stop (Best Practice)
# --restart on-failure: Поднимать только при ошибке (код != 0)

docker run -d --restart unless-stopped nginx
```

### Лимиты (Hard Limits)
Защита хоста от утечек памяти в контейнере. Без этого один Java-контейнер может положить весь сервер.
```bash
docker run -d \
  --memory="512m" \       # OOM Kill при превышении
  --cpus="1.5" \          # Не более 1.5 ядер CPU
  nginx
```

## 3. Логирование и Мониторинг

### Работа с логами
```bash
# Следить в реальном времени (tail -f)
docker logs -f my-app

# Последние 100 строк с таймстемпами
docker logs --tail 100 -t my-app

# Фильтрация по времени (полезно для разбора инцидентов)
docker logs --since 10m my-app  # Логи за последние 10 минут
```

### Ротация логов (Log Rotation)
*Критично для Prod!* По умолчанию Docker пишет JSON-файл бесконечно.
```bash
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
```

### Живой мониторинг
```bash
# Статистика потребления CPU/RAM/NET в реальном времени
docker stats

# Список процессов внутри контейнера (аналог ps aux)
docker top my-app
```

## 4. Инспекция и Метаданные (Inspect)

Использование Go-templates для извлечения данных без парсинга всего JSON глазами.

```bash
# Получить внутренний IP адрес
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-app

# Получить PID процесса на хосте (для профилирования strace/perf)
docker inspect -f '{{.State.Pid}}' my-app

# Узнать, куда смонтированы вольюмы
docker inspect -f '{{ json .Mounts }}' my-app
```

## 5. Обслуживание и Cleanup

### Работа с файлами (cp)
```bash
# Вытащить конфиг из контейнера (бэкап)
docker cp my-app:/app/config.json ./config.json

# Закинуть файл внутрь (hot fix без пересборки)
docker cp ./index.html my-app:/usr/share/nginx/html/
```

### Diff (Аудит изменений)
Показывает файлы, измененные в RW-слое (A-Added, C-Changed, D-Deleted). Помогает понять, почему контейнер "распух".
```bash
docker diff my-app
```

### Массовая очистка
```bash
# Удалить все остановленные контейнеры
docker container prune -f

# Удалить ВСЕ контейнеры (Hard Reset)
docker rm -f $(docker ps -aq)
```
