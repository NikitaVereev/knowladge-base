---
title: "Логи и отладка контейнеров"
description: "Руководство по диагностике контейнеров: просмотр логов, мониторинг ресурсов, инспекция конфигурации и выполнение команд внутри."
---


Если контейнер работает не так, как ожидается, Docker предоставляет набор инструментов для диагностики.

## 1. Просмотр логов (STDOUT/STDERR)

Docker перехватывает стандартный вывод процесса (PID 1) внутри контейнера.

### Основные команды
```bash
# Все логи с момента запуска
docker logs my-app

# Следить за логами в реальном времени (как tail -f)
docker logs -f my-app

# Показать последние 100 строк
docker logs --tail 100 my-app
```

### Фильтрация и поиск
Поскольку `docker logs` выводит текст в stdout, вы можете использовать стандартные unix-утилиты:
```bash
# Найти ошибки (игнорируя регистр)
docker logs my-app | grep -i error

# Сохранить логи в файл для анализа
docker logs my-app > app_debug.log
```

## 2. Инспекция состояния (Inspect)

`docker inspect` возвращает полную конфигурацию контейнера в JSON. Это полезно для проверки IP-адресов, путей к volumes и переменных окружения.

```bash
# Полный JSON
docker inspect my-app

# Получить IP адрес
docker inspect --format '{{.NetworkSettings.IPAddress}}' my-app

# Получить статус (Running/Exited)
docker inspect --format '{{.State.Status}}' my-app

# Проверить pid процесса на хосте
docker inspect --format '{{.State.Pid}}' my-app
```

## 3. Выполнение команд внутри (Exec)

Если контейнер запущен, вы можете войти внутрь для отладки (проверить файлы, сеть).

### Интерактивный вход (Shell)
```bash
docker exec -it my-app bash
# Если bash нет (например, в Alpine), используйте sh:
docker exec -it my-app sh
```

### Одноразовые команды
```bash
# Проверить переменные окружения, которые видит приложение
docker exec my-app env

# Проверить доступность сети из контейнера
docker exec my-app curl -I https://google.com
```

## 4. Мониторинг ресурсов (Stats)

Если контейнер "тормозит", проверьте потребление CPU и памяти.

```bash
# Live-мониторинг (как top)
docker stats

# Мгновенный снимок (без стриминга)
docker stats --no-stream
```

**Расшифровка полей:**
* `CPU %`: Процент использования ядра (может быть >100% на многоядерных системах).
* `MEM USAGE / LIMIT`: Текущая память vs Лимит (если установлен).
* `NET I/O`: Сетевой трафик (вход/выход).

## 5. Расширенная отладка

Если стандартных средств недостаточно, можно установить утилиты внутри контейнера (но лучше это делать только в dev-окружении, так как при перезапуске они пропадут).

```bash
docker exec -it my-app bash
apt-get update && apt-get install -y iputils-ping net-tools procps
# Теперь доступны ping, netstat, ps
```

## Связанные материалы

- [[devops/docker/tutorials/debug-container|Туториал: Практика отладки на примере MongoDB]]
- [[devops/docker/how-to/manage-lifecycle|Управление жизненным циклом]]
