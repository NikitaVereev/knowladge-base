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

