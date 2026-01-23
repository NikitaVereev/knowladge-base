---
title: "7 Бэкап и восстановление кластера"
description: "Как сохранить состояние Swarm и восстановить его после аварии."
---

Состояние кластера (Raft log) хранится в `/var/lib/docker/swarm`.

## Создание бэкапа

Выполнять на **Manager** ноде.

1.  Остановите Docker (рекомендуется для консистентности):
    ```bash
    systemctl stop docker
    ```
2.  Заархивируйте директорию:
    ```bash
    tar -czvf swarm-backup.tar.gz /var/lib/docker/swarm
    ```
3.  Запустите Docker:
    ```bash
    systemctl start docker
    ```

## Восстановление

Если все менеджеры потеряны.

1.  На новой чистой машине остановите Docker.
2.  Удалите существующую папку swarm (если есть): `rm -rf /var/lib/docker/swarm`
3.  Распакуйте архив: `tar -xzvf swarm-backup.tar.gz -C /`
4.  Инициализируйте новый кластер из этих данных:
    ```bash
    docker swarm init --force-new-cluster
    ```

## Связанные материалы
- [[docs/devops/docker-swarm/explanation/1-architecture|Архитектура Swarm]]
