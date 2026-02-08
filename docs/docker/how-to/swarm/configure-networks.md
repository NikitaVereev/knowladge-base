---
title: "Настройка сетей Swarm"
type: how-to
tags: [docker, swarm, networking, overlay, ingress]
sources:
  docs: "https://docs.docker.com/network/drivers/overlay/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[docker/tutorials/06-swarm-cluster]]"
---


# Настройка сетей Swarm

> **TL;DR:** Overlay-сети связывают контейнеры на разных хостах. Создаются на менеджере,
> автоматически распространяются на воркеры при деплое сервиса.



## Создание Overlay сети

Стандартная сеть для общения сервисов между нодами:
```bash
docker network create --driver overlay my-net
```

### Шифрованная сеть (Encrypted)
Если ваши ноды общаются через публичный интернет (или вы хотите Zero Trust), включите шифрование (IPSec):
```bash
docker network create --driver overlay --opt encrypted my-secure-net
```
*Внимание: Это создает небольшую нагрузку на CPU.*

## Публикация портов (Publish)

### Ingress Mode (По умолчанию)
Порт открывается на всех нодах кластера.
```bash
docker service create -p 80:80 nginx
```

### Host Mode
Порт открывается **только** на той ноде, где запущен контейнер. Полезно, если вы сами балансируете трафик или используете мониторинг.
```bash
docker service create --publish mode=host,target=80,published=80 nginx
```


## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Одна overlay-сеть для всего | Frontend видит БД напрямую | Создать отдельные сети: frontend-net, backend-net |
| Нет `--attachable` для docker run | `could not attach to network` | Overlay по умолчанию только для сервисов. `--attachable` для standalone контейнеров |
| Порты 4789/7946 закрыты файрволом | Overlay не работает между хостами | Открыть: TCP/UDP 7946 (gossip), UDP 4789 (VXLAN) |
| Encryption без оценки overhead | Скорость сети упала на 30% | `--opt encrypted` шифрует VXLAN. Использовать только для sensitive трафика |

