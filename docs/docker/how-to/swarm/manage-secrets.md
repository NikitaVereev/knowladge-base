---
title: "Управление секретами Swarm"
type: how-to
tags: [docker, swarm, secrets, security, encryption]
sources:
  docs: "https://docs.docker.com/engine/swarm/secrets/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---

# Управление секретами

> **TL;DR:** `docker secret create` → секрет шифруется в Raft, доставляется в tmpfs
> контейнера (`/run/secrets/`). Безопаснее ENV: не виден в inspect, не на диске.

Никогда не передавайте пароли через переменные окружения (`-e PASSWORD=...`), так как они видны в `docker inspect`. Используйте Docker Secrets.

## Создание секрета

Лучший способ (через stdin, чтобы не оставлять следов в bash history):
```bash
echo "my-super-secret-password" | docker secret create db_pass -
```

Из файла (например, SSL сертификат):
```bash
docker secret create site_cert ./fullchain.pem
```

## Использование в сервисе

Секреты монтируются в контейнер как **файлы** в директорию `/run/secrets/`.

```bash
docker service create \
  --name db \
  --secret db_pass \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_pass \
  postgres
```

*Обратите внимание: Большинство официальных образов (Postgres, MySQL, WordPress) поддерживают суффикс `_FILE` для переменных окружения.*

## Ротация секретов

Секреты неизменяемы. Чтобы обновить пароль:
1.  Создайте новый секрет: `docker secret create db_pass_v2 -`
2.  Обновите сервис:
    ```bash
    docker service update \
      --secret-rm db_pass \
      --secret-add db_pass_v2 \
      db
    ```


## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Секрет в ENV вместо Secret | Виден в `docker inspect`, может утечь в логи | Использовать Docker Secrets + суффикс `_FILE` |
| Изменили секрет — сервис не обновился | Старое значение в контейнере | Секреты иммутабельны. Создать новый, обновить сервис |
| `docker secret` вне Swarm mode | `This node is not a swarm manager` | Secrets работают только в Swarm. `docker swarm init` |
| Права доступа к /run/secrets/ | `Permission denied` при чтении | Секреты по умолчанию доступны root. Указать `uid/gid/mode` в compose |