---
title: 04 Stateful Services
---

---

## Global vs Replicated

```
REPLICATED (по умолчанию):
┌─────────────────────────────────┐
│ replicas: 3                     │
│  ├─ Task 1 (node1)             │
│  ├─ Task 2 (node2)             │
│  └─ Task 3 (node3)             │
└─────────────────────────────────┘

GLOBAL (одна на каждой ноде):
┌─────────────────────────────────┐
│ mode: global                    │
│  ├─ Task (node1)               │
│  ├─ Task (node2)               │
│  ├─ Task (node3)               │
│  ├─ Task (node4)               │
│  └─ Task (node5)               │
└─────────────────────────────────┘
```

---

## Создание Stateful Services

```bash
# Replicated с constraints
docker service create \
  --name db \
  --replicas 1 \
  --constraint node.role==manager \
  --mount type=volume,source=db-data,target=/var/lib/postgresql \
  postgres

# Global сервис (логирование, мониторинг)
docker service create \
  --name monitoring \
  --mode global \
  promtail
```

---

## Volumes для State

```bash
# Named volume
docker service create \
  --name db \
  --mount type=volume,source=db-data,target=/data \
  postgres

# Проверить volume
docker volume ls
docker volume inspect db-data

# Данные сохраняются при перезагрузке контейнера
```

---

## Constraints (Размещение)

```bash
# На manager нодах только
docker service create \
  --constraint node.role==manager \
  --name db \
  postgres

# На worker нодах только
docker service create \
  --constraint node.role==worker \
  --name app \
  myapp

# На нодах с меткой production
docker service create \
  --constraint node.labels.env==production \
  --name web \
  nginx

# Установить метку на ноде
docker node update --label-add env=production node1
```

---

## Практические Примеры

**PostgreSQL:**
```bash
docker service create \
  --name postgres \
  --replicas 1 \
  --constraint node.role==manager \
  --mount type=volume,source=postgres-data,target=/var/lib/postgresql/data \
  --env POSTGRES_PASSWORD=secret \
  postgres:15
```

**Redis:**
```bash
docker service create \
  --name redis \
  --replicas 1 \
  --mount type=volume,source=redis-data,target=/data \
  redis:7
```

**Logging (global на всех нодах):**
```bash
docker service create \
  --name filebeat \
  --mode global \
  --mount type=bind,source=/var/lib/docker/containers,target=/var/lib/docker/containers,readonly=true \
  elastic/filebeat
```

---

## Monitoring State

```bash
# Просмотреть volume на ноде
docker volume inspect db-data

# Логирование на все реплики
docker service logs postgres

# Список нод и их storage
docker node ls -q | xargs docker node inspect
```

---

**Следующее:** [[05-healthchecks|Health Checks и Self-Healing]]
