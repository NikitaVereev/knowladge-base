---
title: 05 Health Checks и Self-Healing
---

---

## HEALTHCHECK в Dockerfile

```dockerfile
FROM nginx

# Health check каждые 10 сек, timeout 5 сек
HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

---

## Health Status

```bash
# Проверить статус контейнера
docker ps

# Статусы:
# healthy - отвечает на healthcheck
# unhealthy - не отвечает
# starting - еще проверяет

# Детали
docker inspect <container> | grep -A 5 Health
```

---

## Service Restart Policy

```bash
# Restart при ошибке
docker service create \
  --name web \
  --restart-condition on-failure \
  --restart-delay 10s \
  nginx

# any - перезагружать всегда
# failure - только при ошибке
# none - не перезагружать

docker service create \
  --restart-condition any \
  --restart-delay 5s \
  --restart-max-attempts 3 \
  app
```

---

## Self-Healing в Swarm

```
1. Container fails
    ↓
2. Manager detects unhealthy
    ↓
3. Automatically restart
    ↓
4. Task moved to healthy node if needed
    ↓
5. Service running again
```

---

## Пример с Docker Stack

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3

  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure

volumes:
  db-data:
```

---

## Monitoring Health

```bash
# Список сервисов и их статус
docker service ps web

# Логи (видны ошибки healthcheck)
docker service logs web

# Частые проверки (custom)
watch -n 1 'docker service ps web'
```

---

**Следующее:** [[06-docker-stack|Docker Stack Deployment]]
