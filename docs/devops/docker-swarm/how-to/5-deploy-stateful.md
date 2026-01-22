---
title: "5 Запуск Stateful приложений (БД)"
description: "Особенности запуска баз данных в Swarm: привязка к ноде и тома."
---

Stateful приложения (БД) сложнее stateless, так как данные лежат на диске конкретной ноды.

## Стратегия 1: Привязка к ноде (Pinning)

Жестко привяжите сервис к конкретной ноде, где лежат данные.

```yaml
services:
  db:
    image: postgres
    volumes:
      - /opt/data/pg:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.hostname == node-1]
```

*   **Плюс:** Простота, максимальная скорость диска (I/O).
*   **Минус:** Если `node-1` сгорит, сервис не переедет, пока вы не восстановите данные на другой ноде.

## Стратегия 2: Сетевые хранилища (NFS / Cloud Volumes)

Используйте драйверы томов, которые монтируют сетевые диски.

```yaml
volumes:
  db-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/to/nfs/share"
```

*   **Плюс:** Сервис может переехать на любую ноду, данные "едут" за ним.
*   **Минус:** Задержки сети, сложность настройки NFS/GlusterFS/Ceph.

## Связанные материалы
- [[devops/docker/explanation/volumes-vs-binds|Тома и Bind Mounts]]
- [[devops/docker-swarm/how-to/manage-placement|Управление размещением]]
