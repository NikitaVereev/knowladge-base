---
title: 02 Overlay Networking и Service Discovery
---

---

## Overlay Network

```
┌────────────────────────────────────┐
│  Overlay Network (10.0.9.0/24)    │
├────────────────────────────────────┤
│                                    │
│  Service 1    Service 2    Service 3
│  (VIP)        (VIP)        (VIP)
│  10.0.9.2     10.0.9.3     10.0.9.4
│      ↓            ↓            ↓
│  Task1 Task2  Task3 Task4  Task5
│  (nginx)      (db)         (cache)
│                                    │
└────────────────────────────────────┘
```

---

## Создание Overlay Network

```bash
# Создать overlay сеть
docker network create --driver overlay my-network

# Проверить
docker network ls | grep overlay

# Детали сети
docker network inspect my-network
```

---

## Service Discovery (DNS)

```bash
# Каждый сервис получает DNS имя
# Пример: web.overlay

# Внутри контейнера (DNS на 127.0.0.11:53)
nslookup web
# Returns: 10.0.9.2 (VIP)

# Ping другого сервиса
docker run --network my-network alpine ping db
# OK!

# Load balancing (DNS возвращает один IP, но LB распределяет)
```

---

## VIP (Virtual IP)

```bash
# Service получает один VIP
docker service inspect web | grep VirtualIPs

# VIP остаётся стабильным даже если:
# - Контейнеры перезагружаются
# - Контейнеры перемещаются на другие ноды
# - Количество реплик меняется

# Load balancing происходит на VIP
# Каждый запрос идёт на разный контейнер
```

---

## Сеть в Swarm

```bash
# Для разных сервисов - разные сети
docker service create --network frontend web
docker service create --network backend db

# frontend и backend не видят друг друга
# Используй reverse proxy для связи

# Или один network для всех (менее безопасно)
docker service create --network shared web
docker service create --network shared db
```

---

## Port Publishing

```bash
# Опубликовать порт (на всех нодах!)
docker service create \
  --publish 8080:80 \
  --name web \
  nginx

# Accessible на ЛЮБОЙ ноде:
curl http://manager:8080
curl http://worker1:8080
curl http://worker2:8080

# Ingress load balancer распределяет
```

---

## Service vs Container DNS

**Container DNS:** 127.0.0.11:53
- Разрешает имена сервисов
- Возвращает VIP
- Load балансер на хосте распределяет

**Пример:**
```bash
# Внутри контейнера web, проверить
cat /etc/resolv.conf
# nameserver 127.0.0.11

# Resolve service
nslookup db
# Server:     127.0.0.11
# Address:    10.0.9.3
```

---

## Multi-Tier Application Example

```bash
# Создать сети
docker network create --driver overlay frontend
docker network create --driver overlay backend

# Frontend tier (публичный)
docker service create \
  --network frontend \
  --publish 8080:80 \
  --name nginx-lb \
  nginx

# App tier (private)
docker service create \
  --network frontend \
  --network backend \
  --name app \
  myapp

# Database tier (private)
docker service create \
  --network backend \
  --name db \
  postgres

# App может access nginx (frontend) и db (backend)
# Nginx может access app через frontend network
# Db isolated в backend network
```

---

## Debugging Networks

```bash
# Список networks
docker network ls

# Информация о network
docker network inspect my-network

# Service в какой сети
docker service inspect web | grep Networks

# Container в какой сети
docker container inspect task | grep Networks

# Test connectivity
docker run --network my-network alpine nslookup web
```

---

**Следующее:** [[03-secrets-configs|Secrets и Configs]]
