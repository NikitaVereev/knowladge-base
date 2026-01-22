---
title: "3 Настройка сетей"
description: "Создание Overlay сетей, шифрование трафика и публикация портов."
---

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

## Связанные материалы
- [[2-networking-concepts|Как работает сеть Swarm (VIP, DNS)]]
- [[9-debug-networking|Отладка сетей]]
