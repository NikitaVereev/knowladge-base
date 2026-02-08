---
title: "04. Сети Docker на практике"
type: tutorial
tags: [docker, tutorial, networking, bridge, dns, service-discovery, isolation]
sources:
  docs: "https://docs.docker.com/network/"
  book: "Docker Deep Dive — Nigel Poulton, Ch.11-12"
related:
  - "[[docker/explanation/networking]]"
  - "[[docker/how-to/local-networking]]"
  - "[[docker/how-to/debug-containers]]"
  - "[[docker/reference/cheatsheet]]"
prev: "[[docker/tutorials/03-compose-app]]"
next: "[[docker/tutorials/05-debugging-workshop]]"
---

# Сети Docker на практике

> **Цель:** Понять как контейнеры общаются друг с другом и с внешним миром.
> После урока ты сможешь проектировать сетевую топологию для любого проекта:
> изолировать сервисы, настраивать DNS, отлаживать сетевые проблемы.

## Предварительные требования

- Пройден [[docker/tutorials/03-compose-app]]
- Установлен Docker (см. [[docker/how-to/install]])

## Зачем это нужно

В уроке 03 ты запускал App + DB + Redis через Compose, и они «магически» видели друг друга по имени. Но что если:
- Нужно изолировать frontend от базы данных?
- Два проекта на одной машине конфликтуют портами?
- Контейнер не может достучаться до другого, и непонятно почему?

Всё это — вопросы сетей. Docker создаёт виртуальные сети, и понимание их устройства — разница между «работает на моей машине» и «работает предсказуемо».

## Шаг 1: Default bridge — почему он плохой

Docker создаёт сеть `bridge` по умолчанию. Посмотрим на неё:

```bash
# Список всех сетей Docker
docker network ls
```

> **Ожидаемый результат:**
> ```
> NETWORK ID     NAME      DRIVER    SCOPE
> a1b2c3d4e5f6   bridge    bridge    local
> f6e5d4c3b2a1   host      host      local
> 1a2b3c4d5e6f   none      null      local
> ```

Запустим два контейнера в default bridge:

```bash
# Контейнер 1
docker run -d --name web1 nginx:alpine

# Контейнер 2
docker run -d --name web2 nginx:alpine
```

Попробуем обратиться из web1 к web2 по имени:

```bash
docker exec web1 ping -c 2 web2
```

> **Ожидаемый результат:** `ping: bad address 'web2'` — **не работает!**

В default bridge нет DNS. Контейнеры могут общаться только по IP:

```bash
# Узнаём IP web2
docker inspect web2 --format '{{.NetworkSettings.IPAddress}}'
# Допустим, 172.17.0.3

# Пингуем по IP — работает
docker exec web1 ping -c 2 172.17.0.3
```

> **Проблема:** IP-адреса динамические. При перезапуске контейнера IP изменится,
> и все хардкоженные адреса сломаются. Поэтому default bridge не подходит
> для production — и даже для разработки.

Уберём за собой:

```bash
docker rm -f web1 web2
```

## Шаг 2: User-defined bridge — правильный подход

Создаём свою сеть:

```bash
# Создаём сеть
docker network create myapp
```

Запускаем контейнеры в этой сети:

```bash
docker run -d --name web1 --network myapp nginx:alpine
docker run -d --name web2 --network myapp nginx:alpine
```

Теперь пробуем пинг по имени:

```bash
docker exec web1 ping -c 2 web2
```

> **Ожидаемый результат:**
> ```
> PING web2 (172.18.0.3): 56 data bytes
> 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.089 ms
> ```
> **Работает!** Docker встроенный DNS (127.0.0.11) автоматически резолвит имена контейнеров.

### Почему user-defined bridge лучше

| Возможность | Default bridge | User-defined bridge |
|-------------|---------------|-------------------|
| DNS по имени контейнера | ✗ | ✓ |
| Автоматическая изоляция | ✗ (все контейнеры видят друг друга) | ✓ (только в своей сети) |
| Подключение/отключение на лету | ✗ | ✓ |
| Настраиваемая подсеть | ✗ | ✓ |

Уберём за собой:

```bash
docker rm -f web1 web2
docker network rm myapp
```

## Шаг 3: Изоляция через множественные сети

Реальный сценарий: frontend не должен иметь прямого доступа к базе данных. Только backend общается с БД.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   frontend   │────│   backend   │────│  database    │
│  (Nginx)    │    │  (Node.js)  │    │ (PostgreSQL) │
└─────────────┘     └─────────────┘     └─────────────┘
      │                    │                    │
   frontend-net      frontend-net          backend-net
                     backend-net
```

Backend подключён к обеим сетям. Frontend и database — каждый только к своей.

```bash
# Создаём две сети
docker network create frontend-net
docker network create backend-net

# Database — только в backend-net
docker run -d \
  --name database \
  --network backend-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# Backend — в обеих сетях
docker run -d \
  --name backend \
  --network backend-net \
  nginx:alpine

# Подключаем backend ко второй сети
docker network connect frontend-net backend

# Frontend — только в frontend-net
docker run -d \
  --name frontend \
  --network frontend-net \
  nginx:alpine
```

Проверяем изоляцию:

```bash
# Frontend → Backend: РАБОТАЕТ (общая frontend-net)
docker exec frontend ping -c 2 backend

# Backend → Database: РАБОТАЕТ (общая backend-net)
docker exec backend ping -c 2 database

# Frontend → Database: НЕ РАБОТАЕТ (нет общей сети)
docker exec frontend ping -c 2 database
```

> **Ожидаемый результат последней команды:** `ping: bad address 'database'`
> Frontend не может даже резолвить имя database — полная изоляция.

Уберём за собой:

```bash
docker rm -f frontend backend database
docker network rm frontend-net backend-net
```

## Шаг 4: Публикация портов — как это работает

Когда ты пишешь `-p 8080:80`, Docker делает следующее:

1. Создаёт правило iptables (DNAT) на хосте
2. Запускает `docker-proxy` процесс
3. Трафик на `host:8080` → перенаправляется на `container:80`

```bash
# Запускаем Nginx с публикацией порта
docker run -d --name web -p 8080:80 nginx:alpine

# Проверяем — доступен на хосте
curl -s http://localhost:8080 | head -5

# Смотрим правила iptables (что Docker создал)
sudo iptables -t nat -L DOCKER -n 2>/dev/null | grep 8080 || echo "(нужен sudo для просмотра iptables)"

# Смотрим процесс docker-proxy
ps aux | grep docker-proxy | grep -v grep || echo "(docker-proxy может не отображаться в контейнере)"
```

### Варианты публикации

```bash
# Конкретный порт хоста → порт контейнера
docker run -d -p 8080:80 nginx:alpine

# Случайный порт хоста (Docker выберет сам)
docker run -d -p 80 nginx:alpine
# Узнать назначенный порт:
docker port <container_id>

# Только на localhost (не доступен извне)
docker run -d -p 127.0.0.1:8080:80 nginx:alpine

# Только на конкретный интерфейс
docker run -d -p 192.168.1.100:8080:80 nginx:alpine

# UDP порт
docker run -d -p 5353:53/udp dns-server
```

> **Важно:** `-p 127.0.0.1:8080:80` — безопасный вариант для разработки.
> Без указания адреса Docker откроет порт на **всех** интерфейсах (0.0.0.0),
> включая публичный IP сервера.

Уберём за собой:

```bash
docker rm -f web
```

## Шаг 5: Сети в Docker Compose

В Compose всё то же самое, но декларативно. Compose автоматически создаёт сеть `<project>_default`, но мы можем определить свои:

Создай файл `compose.yaml`:

```yaml
services:
  frontend:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend-net
    depends_on:
      - backend

  backend:
    image: nginx:alpine
    networks:
      - frontend-net
      - backend-net
    depends_on:
      - database

  database:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend-net

networks:
  frontend-net:
    # Docker создаст bridge-сеть с именем <project>_frontend-net
  backend-net:
    # Отдельная сеть — database изолирована от frontend

volumes:
  db-data:
```

```bash
# Запускаем
docker compose up -d

# Проверяем созданные сети
docker network ls | grep -E "frontend|backend"

# Проверяем: frontend видит backend, но не видит database
docker compose exec frontend ping -c 1 backend    # ✓ работает
docker compose exec frontend ping -c 1 database   # ✗ не работает

# Останавливаем
docker compose down
```

### Кастомная подсеть

Если нужен конкретный диапазон IP (например, чтобы не конфликтовать с VPN):

```yaml
networks:
  backend-net:
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

## Шаг 6: Диагностика сетевых проблем

Набор команд, который спасает при сетевых проблемах:

```bash
# Какие сети существуют
docker network ls

# Подробности о сети: подсеть, gateway, подключённые контейнеры
docker network inspect frontend-net

# К каким сетям подключён контейнер
docker inspect backend --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool

# DNS резолвинг изнутри контейнера
docker exec backend nslookup database

# Проверка связности
docker exec backend ping -c 2 database

# Тяжёлая артиллерия — netshoot (контейнер с сетевыми утилитами)
docker run --rm -it --network frontend-net nicolaka/netshoot
# Внутри netshoot: dig, nslookup, tcpdump, iperf, curl, nmap и т.д.
```

## Проверка результата

Ты должен уметь ответить на вопросы:

```bash
# 1. Создай две сети и три контейнера так, чтобы:
#    - api видел и db, и cache
#    - db и cache НЕ видели друг друга
#    Подсказка: нужны 2 сети, api подключён к обеим

# 2. Запусти Nginx доступный ТОЛЬКО с localhost:
docker run -d -p 127.0.0.1:9090:80 --name test nginx:alpine
curl http://localhost:9090    # ✓
# С другой машины по IP сервера — не откроется
docker rm -f test
```

## Контрольные вопросы

- Почему в default bridge не работает DNS, а в user-defined работает?
- Как сделать так, чтобы контейнер A видел B, но не видел C?
- Что произойдёт, если два контейнера в разных Compose-проектах захотят общаться?
- Чем `-p 8080:80` отличается от `-p 127.0.0.1:8080:80` с точки зрения безопасности?

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Используют default bridge | Контейнеры не видят друг друга по имени | Создать user-defined network |
| `-p 8080:80` на сервере | Порт открыт для всего интернета | Использовать `-p 127.0.0.1:8080:80` + reverse proxy |
| Забыли `--network` при `docker run` | Контейнер в default bridge, не видит остальные | Добавить `--network <name>` |
| Конфликт подсетей с VPN | Контейнеры не получают IP, сеть не создаётся | Задать кастомную подсеть в `ipam.config` |
| UFW не блокирует Docker-порт | Docker вставляет iptables-правила ДО UFW | Использовать цепочку `DOCKER-USER` |

## Что дальше

→ [[docker/tutorials/05-debugging-workshop]] — применим знания о сетях для отладки реальных проблем

**Хочешь глубже?**
- Теория: [[docker/explanation/networking]] — CNM, veth pairs, iptables, overlay
- Практика: [[docker/how-to/local-networking]] — host.docker.internal, SSH forwarding
