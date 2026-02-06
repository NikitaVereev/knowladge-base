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

