---
title: "Настройка Healthchecks"
type: how-to
tags: [docker, swarm, healthcheck, monitoring, readiness]
sources:
  docs: "https://docs.docker.com/engine/reference/builder/#healthcheck"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---

# Healthchecks в Swarm

> **TL;DR:** Healthcheck в Swarm определяет, когда контейнер готов принимать трафик.
> Без healthcheck Swarm не может откатить неудачное обновление.



Swarm умеет перезапускать зависшие контейнеры, если настроен `HEALTHCHECK`.

## Пример в Compose

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s       # Как часто проверять
      timeout: 10s        # Таймаут ожидания ответа
      retries: 3          # Сколько раз ошибиться перед рестартом
      start_period: 40s   # Время на прогрев приложения (ошибки игнорируются)
```

## Как это работает?
1.  Docker внутри контейнера выполняет команду `test`.
2.  Если она возвращает `exit code 0` — статус `healthy`.
3.  Если `exit code 1` три раза подряд (`retries`) — статус `unhealthy`.
4.  Swarm убивает контейнер и запускает новый.


## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Нет healthcheck | Rolling update деплоит сломанную версию на все реплики | Всегда добавлять healthcheck для production-сервисов |
| `start_period` слишком короткий | Контейнер unhealthy при старте → Swarm перезапускает бесконечно | Увеличить `start_period` (30s для Java/Spring) |
| Healthcheck проверяет внешний сервис | БД упала → все контейнеры приложения unhealthy → cascading failure | Проверять только локальный процесс, не зависимости |
| `curl` в Alpine | `curl: not found` | Использовать `wget --spider` (есть в Alpine) |