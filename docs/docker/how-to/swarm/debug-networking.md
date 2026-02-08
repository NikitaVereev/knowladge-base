---
title: "Отладка сети Swarm"
type: how-to
tags: [docker, swarm, networking, debug, troubleshooting, overlay]
sources:
  docs: "https://docs.docker.com/network/network-tutorial-overlay/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---


## 1. Проверка DNS (netshoot)

Запустите утилиту `netshoot` в той же сети, где находится проблемный сервис.

```bash
docker service create --name debug --network my-net --mode global nicolaka/netshoot sleep infinity
```

Зайдите в контейнер:
```bash
# Найдите ID контейнера на текущей ноде

> **TL;DR:** Overlay не работает? Проверь: порты 7946+4789 открыты, ноды видят друг друга,
> `docker network inspect` показывает peers. Netshoot — для глубокой диагностики.
docker ps | grep debug
docker exec -it <container-id> zsh
```

Внутри проверьте резолвинг имени сервиса:
```bash
nslookup my-web-service
dig my-web-service
```

## 2. Проверка портов

Проверьте, открыты ли необходимые порты между нодами (Firewall):
*   **TCP/2377**: Cluster Management
*   **TCP/UDP/7946**: Communication among nodes
*   **UDP/4789**: Overlay Network Traffic (VXLAN)

Если 4789 закрыт, сервисы будут создаваться, но `ping` между ними ходить не будет.


## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Файрвол блокирует VXLAN (UDP 4789) | Контейнеры на разных нодах не видят друг друга | Открыть TCP/UDP 7946 + UDP 4789 между всеми нодами |
| DNS не резолвится | `Could not resolve host: myservice` | Проверить: сервис в той же overlay-сети? `docker service inspect` |
| VIP не отвечает | Сервис `healthy`, но трафик не идёт | Проверить `docker network inspect` → IPAM, Peers. Перезапустить `ingress` |
| Netshoot не подключается к overlay | `could not attach to network` | Overlay должна быть `--attachable` для standalone контейнеров |

