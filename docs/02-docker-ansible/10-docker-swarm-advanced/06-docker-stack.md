---
title: 06 Docker Stack Deployment
---

---

## Docker Stack vs Service

```
docker service create    → одиночный сервис
docker stack deploy      → несколько сервисов
                          + networks + volumes
                          из compose файла
```

---

## Stack Развёртывание

```bash
# Развернуть stack
docker stack deploy -c docker-compose.yml myapp

# Список stacks
docker stack ls

# Сервисы в stack
docker stack services myapp

# Задачи в stack
docker stack ps myapp

# Логи stack
docker stack logs myapp

# Удалить stack
docker stack rm myapp
```

---

## Полный docker-compose.yml для Swarm

```yaml
services:
  # Frontend
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - frontend
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Application
  app:
    image: myapp:latest
    environment:
      DB_HOST: db
      DB_PASSWORD_FILE: /run/secrets/db_password
    networks:
      - frontend
      - backend
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s
    secrets:
      - db_password
    depends_on:
      - db

  # Database
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay

volumes:
  db-data:
    driver: local

secrets:
  db_password:
    external: true
```

---

## Update Strategy

```yaml
deploy:
  update_config:
    parallelism: 1        # Обновлять по одному
    delay: 10s            # Ждать 10 сек между
    failure_action: pause # При ошибке паузу
```

---

## Monitoring Stack

```bash
# Status
docker stack ps myapp

# Логи
docker stack logs myapp -f

# Service details
docker service inspect <service-name>

# Network
docker network ls | grep myapp
```

---

**Следующее:** [[07-fault-tolerance|Fault Tolerance и Мониторинг]]
