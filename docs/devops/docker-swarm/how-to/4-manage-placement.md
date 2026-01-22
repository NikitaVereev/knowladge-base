---
title: "Управление размещением (Placement)"
description: "Как указать Swarm, где запускать контейнеры (Labels & Constraints)."
---

Вы можете управлять тем, на какие ноды попадают ваши сервисы, используя метки и ограничения.

## 1. Добавление меток (Labels) на ноды

Пометьте ноды в соответствии с их характеристиками (SSD, GPU, Zone).

```bash
# На менеджере
docker node update --label-add ssd=true node-1
docker node update --label-add zone=us-east node-2
```

## 2. Использование Constraints (Ограничений)

При создании сервиса укажите требование:

```bash
docker service create \
  --name db \
  --constraint 'node.labels.ssd == true' \
  postgres
```

В файле стека (`docker-compose.yml`):
```yaml
deploy:
  placement:
    constraints:
      - node.role == worker
      - node.labels.ssd == true
```

## 3. Глобальные ограничения

Популярные встроенные метки:
*   `node.role == manager`
*   `node.hostname == db-server`
*   `node.platform.os == linux`
