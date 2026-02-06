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


