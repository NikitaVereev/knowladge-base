---
title: "Управление приложениями Compose"
description: "Основные команды CLI: up, down, logs, exec. Управление жизненным циклом."
---


## Запуск и остановка

| Команда | Описание |
|---------|----------|
| `docker compose up` | Запустить все сервисы (в консоли) |
| `docker compose up -d` | Запустить в фоновом режиме (Detached) |
| `docker compose up --build` | Пересобрать образы перед запуском |
| `docker compose stop` | Остановить контейнеры (данные в памяти теряются, на диске — нет) |
| `docker compose down` | Остановить и **удалить** контейнеры и сеть |
| `docker compose down -v` | Удалить контейнеры, сеть и **Volumes** (осторожно!) |

## Мониторинг и логи

```bash
# Посмотреть статус контейнеров текущего проекта
docker compose ps

# Читать логи всех сервисов (следить в реальном времени)
docker compose logs -f

# Логи только конкретного сервиса
docker compose logs -f web
```

## Выполнение команд

```bash
# Войти в shell контейнера 'web'
docker compose exec web bash

# Выполнить команду в контейнере 'db'
docker compose exec db psql -U user -d mydb
```

## Ports vs Expose

В `docker-compose.yml` часто путают `ports` и `expose`.

* **ports:** Публикует порт на хост-машине. Доступно из интернета/локальной сети.
  ```yaml
  ports:
    - "8080:80"  # localhost:8080 -> container:80
  ```
* **expose:** Открывает порт только для других сервисов **внутри** сети Docker. С хоста недоступен. Используется для внутренних сервисов (БД, кэш), к которым не нужно подключаться напрямую снаружи.

## Связанные материалы
- [[devops/docker/tutorials/compose-stacks|Примеры конфигураций]]
