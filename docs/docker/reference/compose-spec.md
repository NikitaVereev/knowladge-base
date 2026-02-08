---
title: "Спецификация Docker Compose"
type: reference
tags: [docker, compose, yaml, spec, services, networks, volumes]
sources:
  docs: "https://docs.docker.com/compose/compose-file/"
related:
  - "[[docker/explanation/compose-model]]"
  - "[[docker/explanation/configuration]]"
  - "[[docker/tutorials/03-compose-app]]"
---

# Спецификация Compose v2 (Справочник)

> **Справочник:** Полная спецификация compose.yaml — services, networks, volumes,
> configs, secrets, deploy.

Compose Specification (ранее docker-compose.yml версии 3) — это стандарт описания многоконтейнерных приложений.

## Структура файла `compose.yaml`

Файл состоит из трех верхнеуровневых секций: `services`, `networks` и `volumes`.

```yaml
services:
  # Определение контейнеров
  webapp:
    image: nginx
    ...

networks:
  # Определение сетей
  app-net:
    driver: bridge

volumes:
  # Определение томов
  db-data:
```

## 1. Services (Сервисы)

Ключевые параметры конфигурации сервиса.

| Поле | Описание | Пример |
| :--- | :--- | :--- |
| `image` | Образ для запуска. | `postgres:15-alpine` |
| `build` | Сборка из Dockerfile. | `context: .`, `dockerfile: Dockerfile.prod` |
| `container_name` | Фиксированное имя контейнера (не рекомендуется для масштабирования). | `my-db` |
| `ports` | Публикация портов (Host:Container). | `["8080:80"]` |
| `expose` | Открыть порт только для других сервисов (не на хост). | `["5432"]` |
| `volumes` | Монтирование томов или путей. | `["./data:/var/lib/mysql"]` |
| `environment` | Переменные окружения (список или словарь). | `DB_HOST: postgres` |
| `env_file` | Файл с переменными (.env). | `["./.env"]` |
| `depends_on` | Зависимости запуска. | `db: { condition: service_healthy }` |
| `restart` | Политика перезапуска. | `unless-stopped` |
| `command` | Переопределение CMD образа. | `["npm", "run", "dev"]` |
| `entrypoint` | Переопределение ENTRYPOINT образа. | `["/app/init.sh"]` |
| `user` | UID:GID запуска. | `1000:1000` |
| `profiles` | Группировка сервисов для выборочного запуска. | `["debug", "tools"]` |

### Healthcheck (Проверка здоровья)
Критически важно для `depends_on`.

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s # Время на "разогрев"
```

### Logging (Логирование)
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

## 2. Networks (Сети)

По умолчанию Compose создает одну сеть `default`.

```yaml
networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
```

Подключение сервиса к сети:
```yaml
services:
  app:
    networks:
      - backend
      - frontend
```

### Статические IP (редко нужно)
```yaml
networks:
  my-net:
    ipam:
      config:
        - subnet: 172.20.0.0/16

services:
  app:
    networks:
      my-net:
        ipv4_address: 172.20.0.5
```

## 3. Volumes (Тома)

```yaml
volumes:
  # Создать новый пустой том
  db-data:
  
  # Использовать уже созданный внешний том
  external-data:
    external: true
```

## 4. Различия v2 и v3 (Legacy)

В современной спецификации (Compose Spec):
*   Поле `version: "3.8"` **больше не нужно**.
*   Команда `docker-compose` (Python, v1) заменена на `docker compose` (Go, v2).
*   Параметры ресурсов (`cpus`, `memory`) переехали из `deploy` (Swarm) напрямую в сервис (для локального Docker).

### Лимиты ресурсов (Deploy)
Работает и в Swarm, и в Docker Compose v2.

```yaml
deploy:
  resources:
    limits:
      cpus: '0.50'
      memory: 512M
    reservations:
      cpus: '0.25'
      memory: 128M
```