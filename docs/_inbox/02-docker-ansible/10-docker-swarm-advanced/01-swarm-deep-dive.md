---
title: 01 Архитектура Swarm (Deep Dive)
---

---

## Manager vs Worker

```
┌──────────────────────────────────────────────┐
│         Docker Swarm Cluster                 │
├──────────────────────────────────────────────┤
│                                              │
│  MANAGERS (управление, состояние):          │
│  ┌─────────────┐  ┌─────────────┐            │
│  │  Manager 1  │  │  Manager 2  │  ...      │
│  │  (Leader)   │  │             │            │
│  └─────────────┘  └─────────────┘            │
│         ↓ Raft Consensus Database            │
│  ┌──────────────────────────┐                │
│  │  Distributed State DB    │                │
│  │  • Services              │                │
│  │  • Tasks                 │                │
│  │  • Networks              │                │
│  └──────────────────────────┘                │
│                                              │
│  WORKERS (выполнение контейнеров):          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │Worker 1 │  │Worker 2 │  │Worker 3 │     │
│  │Containers  │Containers  │Containers      │
│  └─────────┘  └─────────┘  └─────────┘     │
│                                              │
└──────────────────────────────────────────────┘
```

---

## Raft Consensus

**Как это работает:**
- Все managers имеют копию состояния
- Только leader может изменять состояние
- При изменении leader отправляет всем followers
- Followers подтверждают изменение
- Изменение считается committed когда большинство согласны

**Quorum (кворум):**
```
N managers → нужно (N/2 + 1) для quorum

1 manager → 1
3 managers → 2
5 managers → 3
7 managers → 4
```

**Правило:** всегда нечётное число managers!

---

## Состояние Swarm

```bash
# Просмотр всех нод
docker node ls

# Информация о ноде
docker node inspect node1

# Информация о swarm
docker info | grep Swarm
```

---

## Service Scheduling

```
Service Definition
    ├─ replicas: 3
    ├─ image: nginx
    ├─ ports: 8080:80
    └─ constraints: [node.role==worker]
         ↓
    Scheduler (на manager)
         ↓
    Распределение на worker'ы
         ↓
    3 Task на разных worker'ах
         ↓
    3 Running Containers
```

---

## Leader Election

```bash
# Leader выбирается автоматически
# Если leader падает:
1. Followers detekt его отсутствие
2. Проводят election
3. Новый leader выбран за ~10 сек
4. Cluster продолжает работать

# Если leader вернулся:
- Синхронизирует состояние
- Становится follower
```

---

## Fault Tolerance

**При падении 1 из 3 managers:**
```
Было: 3 managers (quorum = 2)
Пало: 1 manager
Осталось: 2 managers (quorum достаточно)
Результат: Cluster работает нормально ✓
```

**При падении 2 из 3 managers:**
```
Было: 3 managers (quorum = 2)
Пало: 2 managers
Осталось: 1 manager (quorum 2, есть только 1)
Результат: Cluster неработоспособен ✗
Решение: kubectl drain или переотправить 2 manager'а
```

---

## Service Loadbalancing

```
Client → 8080:80

┌────────────────────────────────┐
│  Ingress Load Balancer         │
│  (на каждой ноде)             │
└────────────────────────────────┘
            ↓
    ┌───────┼───────┐
    ↓       ↓       ↓
Task 1   Task 2   Task 3
(nginx)  (nginx)  (nginx)
```

---

## DNS Service Discovery

```
# Каждый сервис получает DNS имя
service_name

# Свой DNS сервер в swarm
127.0.0.11:53

# VIP (Virtual IP)
docker service inspect web | grep VirtualIPs

# Внутри контейнера:
nslookup web
# Returns: VIP адрес (например, 10.0.9.2)
```

---

## Просмотр Состояния

```bash
# Список всех нод
docker node ls

# Информация о конкретной ноде
docker node inspect <node>

# Список всех сервисов
docker service ls

# Задачи в сервисе (на каких нодах)
docker service ps <service>

# Логи сервиса
docker service logs <service>

# Статистика
docker stats
```

---

## Best Practices

✓ **Manager configuration:**
- Минимум 3 managers для HA
- Максимум 7 managers (election медленнее)
- Separate managers от рабочих нагрузок

✓ **Worker management:**
- Drain перед отключением
- Добавлять постепенно
- Мониторить ресурсы

✓ **Backup:**
- Регулярно backup manager DB
- Хранить в safe месте
- Имеет encrypted secrets!

---

**Следующее:** [[02-swarm-networking|Overlay Networking и Service Discovery]]
