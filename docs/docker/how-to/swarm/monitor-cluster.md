---
title: "Мониторинг кластера"
type: how-to
tags: [docker, swarm, monitoring, prometheus, events]
sources:
  docs: "https://docs.docker.com/engine/swarm/admin_guide/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---


# Мониторинг кластера

> **TL;DR:** `docker node ls` — состояние нод. `docker service ps` — состояние задач.
> Prometheus + cAdvisor для метрик, Loki для логов.



## Статус нод

```bash
docker node ls
```
Смотрите на колонки **STATUS** (должно быть `Ready`) и **AVAILABILITY** (должно быть `Active`).

## Статус сервисов

```bash
docker service ls
```
Колонка **REPLICAS** показывает `Actual/Desired` (например, `3/3`). Если видите `0/3`, значит контейнеры не могут запуститься.

## Логирование

Смотреть логи всех реплик сервиса одновременно:
```bash
docker service logs -f my-web-service
```
*Совет: Используйте `--raw` или `--tail 100` для удобства.*

## События

Что происходит в кластере прямо сейчас (рестарты, обновления):
```bash
docker events --types service,node
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Мониторинг только менеджеров | Не видят проблемы на воркерах | cAdvisor/node-exporter как `global` service (на каждой ноде) |
| `docker service logs` на большом кластере | Медленно, таймаут | Централизованное логирование (Loki, ELK) через logging driver |
| Нет алертов на `node down` | Узнали о проблеме от пользователей | Prometheus alert на `node_up == 0` |
