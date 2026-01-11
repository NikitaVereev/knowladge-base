### Docker Compose: Advanced Patterns

Advanced техники и best practices для Docker Compose.

---

#### Build Configuration

**Build from Dockerfile:**

```yaml
services:
  app:
    build: .                              # контекст = текущая папка
    # Полностью эквивалентно:
    image: myapp:latest

  api:
    build:
      context: ./api                      # контекст (где искать файлы)
      dockerfile: Dockerfile.prod         # путь к Dockerfile
      args:
        BUILD_ENV: production
        NODE_VERSION: 18
    image: myapi:1.0
```

**Args (аргументы сборки):**

```dockerfile
# Dockerfile
ARG NODE_VERSION=18
ARG BUILD_ENV=development

FROM node:${NODE_VERSION}-alpine

ENV BUILD_ENV=${BUILD_ENV}
RUN echo "Building for $BUILD_ENV"
```

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      args:
        NODE_VERSION: 18
        BUILD_ENV: production
```

---

#### Зависимости между сервисами

**depends_on (базовая зависимость):**

```yaml
version: '3.9'

services:
  app:
    image: myapp
    depends_on:
      - db              # ждать, чтобы db запустился
      - cache

  db:
    image: postgres

  cache:
    image: redis
```

**depends_on с condition (ждать готовности):**

```yaml
version: '3.9'

services:
  app:
    image: myapp
    depends_on:
      db:
        condition: service_healthy       # ждать healthcheck
      cache:
        condition: service_started       # ждать просто запуска

  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis
```

---

#### Restart Policies

**Политики перезапуска:**

```yaml
services:
  web:
    image: nginx
    restart_policy:
      condition: on-failure
      delay: 5s
      max_attempts: 3
      window: 120s

  db:
    image: postgres
    restart_policy:
      condition: always

  cache:
    image: redis
    restart_policy:
      condition: no                      # не перезапускать
```

**Значения condition:**
- `no` — не перезапускать
- `always` — всегда перезапускать
- `on-failure` — перезапустить если вышел с ошибкой
- `unless-stopped` — перезапустить кроме как если явно остановлен

---

#### Environment Variables

**.env файл:**

```env
# .env (в корне проекта)
DATABASE_URL=postgresql://user:pass@db:5432/myapp
REDIS_URL=redis://cache:6379/0
NODE_ENV=development
API_PORT=8000
```

**Использование в docker-compose.yml:**

```yaml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "${API_PORT}:${API_PORT}"
    environment:
      DATABASE_URL: ${DATABASE_URL}
      REDIS_URL: ${REDIS_URL}
      NODE_ENV: ${NODE_ENV}
    # Или загрузить весь .env файл:
    env_file:
      - .env                             # все переменные из .env
      - .env.local                       # перезаписать локальными

  db:
    image: postgres
    environment:
      POSTGRES_URL: ${DATABASE_URL}
```

**Проверка переменных:**

```bash
docker compose config                   # показать с подставленными переменными
cat .env | grep DATABASE                # проверить .env
```

**Переменные могут быть:**
- Из .env файла
- Из переменных окружения хоста
- Default значения в docker-compose.yml

---

#### Profiles (Selective Startup)

**Разделение сервисов по профилям:**

```yaml
version: '3.9'

services:
  # Основные сервисы (запускаются всегда)
  web:
    image: nginx
    profiles: []                         # пустой = базовый

  api:
    image: myapi
    profiles: []

  db:
    image: postgres
    profiles: []

  # Development сервисы (запускаются с --profile dev)
  adminer:
    image: adminer                       # web UI для БД
    profiles: [dev]
    ports:
      - "8081:8080"
    depends_on:
      - db

  phpmyadmin:
    image: phpmyadmin
    profiles: [dev, debug]
    depends_on:
      - db

  # Testing сервисы
  test_runner:
    image: myapp:test
    profiles: [test]
    command: npm test

  # Monitoring
  prometheus:
    image: prom/prometheus
    profiles: [monitoring, prod]

  grafana:
    image: grafana/grafana
    profiles: [monitoring, prod]
```

**Использование:**

```bash
docker compose up                       # запустить только базовые (web, api, db)

docker compose up --profile dev         # + dev сервисы (adminer)

docker compose up --profile monitoring  # + monitoring (prometheus, grafana)

docker compose up --profile dev --profile monitoring   # + dev и monitoring

COMPOSE_PROFILES=dev,monitoring docker compose up      # через env переменную

docker compose up                       # без --profile не запускаются профилированные сервисы
```

**Запуск одного сервиса с профилем:**

```bash
docker compose run --rm adminer         # запустить даже если в профиле [dev]
```

---

#### Volume Management в Compose

**Named volumes:**

```yaml
version: '3.9'

services:
  db:
    image: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

  api:
    image: myapi
    volumes:
      - shared_data:/app/data

volumes:
  postgres_data:                         # создаст docker volume
    driver: local
  shared_data:
    driver: local
```

**Bind mounts:**

```yaml
services:
  dev:
    image: node:18
    volumes:
      - .:/app                           # текущая папка → /app в контейнере
      - /app/node_modules                # исключить node_modules (anonymous volume)
      - ./config:/app/config:ro          # read-only конфиг
```

**Sharing volumes между сервисами:**

```yaml
services:
  api:
    image: myapi
    volumes:
      - shared:/app/data

  worker:
    image: myworker
    volumes:
      - shared:/worker/data              # тот же volume

volumes:
  shared:
```

---

#### Override Files

**Automatic override (docker-compose.override.yml):**

```yaml
# docker-compose.yml (production)
version: '3.9'
services:
  web:
    image: myapp:1.0
    ports:
      - "80:80"
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secure_prod_password

# docker-compose.override.yml (development, автоматически применяется)
version: '3.9'
services:
  web:
    ports:
      - "3000:3000"                      # development port
    volumes:
      - .:/app                           # bind mount для development
  db:
    environment:
      POSTGRES_PASSWORD: dev_password
```

**Custom override files:**

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

**Пример структуры:**

```
docker-compose.yml                      # base конфиг
docker-compose.dev.yml                  # development overrides
docker-compose.prod.yml                 # production overrides
docker-compose.test.yml                 # test overrides
```

```yaml
# docker-compose.yml (base)
services:
  app:
    image: myapp
    restart_policy:
      condition: on-failure

# docker-compose.dev.yml
services:
  app:
    volumes:
      - .:/app
    environment:
      DEBUG: "true"

# docker-compose.prod.yml
services:
  app:
    restart_policy:
      condition: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
```

---

#### Shared Configurations (DRY)

**YAML якоря и ссылки:**

```yaml
version: '3.9'

# Определить якорь (&)
x-common-variables: &common_vars
  environment:
    LOG_LEVEL: info
    APP_ENV: development

x-healthcheck: &healthcheck
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
    interval: 30s
    timeout: 10s
    retries: 3

services:
  api:
    <<: *common_vars              # вставить (merge)
    <<: *healthcheck
    image: myapi
    ports:
      - "3000:3000"

  worker:
    <<: *common_vars              # вставить
    image: myworker
    command: npm run worker

  cache:
    <<: *healthcheck              # вставить
    image: redis
```

---

#### Health Checks

**Базовый healthcheck:**

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s                      # проверять каждые 30 сек
      timeout: 10s                       # timeout проверки
      retries: 3                         # макс попыток перед unhealthy
      start_period: 40s                  # время до первой проверки
```

**Различные типы тестов:**

```yaml
healthcheck:
  # Exec (наиболее общий)
  test: ["CMD", "curl", "-f", "http://localhost"]
  
  # Shell
  test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]
  
  # Простая команда
  test: ["curl", "-f", "http://localhost"]
```

**Примеры для разных сервисов:**

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U user"]

# MySQL
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]

# MongoDB
healthcheck:
  test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet

# Node.js API
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

---

#### Scaling Services

**Запуск нескольких экземпляров:**

```bash
docker compose up --scale api=3 --scale worker=2
# Запустит 3 api контейнера и 2 worker контейнера
```

**В docker-compose.yml (для production):**

```yaml
version: '3.9'

services:
  api:
    image: myapi
    deploy:
      replicas: 3                        # 3 экземпляра
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  worker:
    image: myworker
    deploy:
      replicas: 2
```

---

#### Практические примеры

##### Пример 1: Development Setup (код синхронизируется)

```yaml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app                           # код синхронизируется
      - /app/node_modules                # node_modules не синхро
    environment:
      NODE_ENV: development
      DEBUG: "true"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: dev_app
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev_pass
    volumes:
      - db_dev:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 10s

  adminer:
    image: adminer
    ports:
      - "8081:8080"
    depends_on:
      - db

volumes:
  db_dev:
```

##### Пример 2: Full Stack Production

```yaml
version: '3.9'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: myapp-frontend:1.0
    ports:
      - "80:80"
    depends_on:
      - backend
    restart_policy:
      condition: always

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: myapp-backend:1.0
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:${DB_PASS}@db:5432/myapp
      REDIS_URL: redis://cache:6379/0
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    restart_policy:
      condition: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart_policy:
      condition: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]

  cache:
    image: redis:7-alpine
    restart_policy:
      condition: always
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  db_data:
```

##### Пример 3: Микросервисная архитектура

```yaml
version: '3.9'

services:
  # API Gateway
  gateway:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api-users
      - api-products
    profiles: []

  # Microservices
  api-users:
    build: ./services/users
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/users
      REDIS_URL: redis://cache:6379/0
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    profiles: []

  api-products:
    build: ./services/products
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/products
      REDIS_URL: redis://cache:6379/1
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    profiles: []

  # Workers
  worker-email:
    build: ./workers/email
    environment:
      MESSAGE_BROKER_URL: amqp://user:pass@rabbitmq:5672
    depends_on:
      - rabbitmq
    profiles: [workers]

  worker-notifications:
    build: ./workers/notifications
    environment:
      MESSAGE_BROKER_URL: amqp://user:pass@rabbitmq:5672
    depends_on:
      - rabbitmq
    profiles: [workers]

  # Infrastructure
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: pass
    volumes:
      - db_data:/var/lib/postgresql/data
    profiles: []

  cache:
    image: redis:7-alpine
    profiles: []

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: user
      RABBITMQ_DEFAULT_PASS: pass
    ports:
      - "15672:15672"              # management UI
    profiles: [workers]

  # Monitoring
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    profiles: [monitoring]

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    profiles: [monitoring]

volumes:
  db_data:
```

**Использование:**

```bash
# Development (только основные сервисы)
docker compose up

# Development с workers
docker compose up --profile workers

# Full stack (все + monitoring)
docker compose up --profile workers --profile monitoring

# Production (только основные без профилей)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

#### Best Practices

✅ **Версионирование:** используй специфичные версии образов (не latest)  
✅ **Healthchecks:** добавляй для всех критичных сервисов  
✅ **depends_on с condition:** проверить готовность, не просто запуск  
✅ **Volumes для данных:** используй named volumes для persistent data  
✅ **Environment переменные:** храни чувствительные данные в .env  
✅ **Profiles:** разделяй dev, prod, monitoring сервисы  
✅ **Override files:** не меняй base compose для разных окружений  
✅ **DRY принцип:** используй YAML якоря для повторяющегося кода  
✅ **Logging:** конфигурируй logging driver  
✅ **Restart policies:** установи для production сервисов  