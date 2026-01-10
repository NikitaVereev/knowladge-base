### Docker Compose: Введение и основы

Управление multi-container приложениями с Docker Compose.

---

#### Что такое Docker Compose

**Docker Compose:**
- Инструмент для определения и запуска multi-container Docker приложений
- Использует файл конфигурации (YAML) для описания сервисов
- Позволяет запустить приложение одной командой
- Автоматически создаёт сети, volumes, контейнеры

**Когда использовать:**
- Development окружение (не нужно запоминать длинные docker run команды)
- Testing (воспроизведение production окружения локально)
- Small production (для простых приложений на одной машине)
- CI/CD (тестирование и deployment)

**Когда НЕ использовать:**
- Большие production приложения (используй Kubernetes, Docker Swarm)
- Multi-host deployment (Docker Compose для одной машины)
- Требуется высокая доступность

---

#### Установка Docker Compose

**Обычно идёт с Docker Desktop:**
```bash
docker compose version          # проверить версию (v2+)
docker-compose version          # старая версия (v1)
```

**Если нужна ручная установка:**
```bash
# На Linux (скачать последнюю версию)
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# На macOS (через Homebrew)
brew install docker-compose

# Проверка
docker compose version
```

---

#### Основы YAML

**Что такое YAML:**
- YAML (YAML Ain't Markup Language) — формат для представления данных
- Основан на отступах (indentation), не на скобках/тегах
- Легко читать и писать людям
- Используется в Docker Compose, Kubernetes, Ansible

**Синтаксис YAML:**

```yaml
# Комментарий

# Строка (без кавычек если нет спецсимволов)
name: docker-app
version: 1.0

# Строка с кавычками (если есть спецсимволы)
description: "This is a: special string"

# Многострочный текст (сохранить перенос строк - |)
script: |
  #!/bin/bash
  echo "Hello"
  echo "World"

# Многострочный текст (объединить в одну строку - >)
summary: >
  This is a long
  text that will be
  joined into one line

# Числа
port: 8080
memory: 1024
cpu: 0.5

# Булевы значения
enabled: true
disabled: false
running: yes
stopped: no

# Список (массив)
ports:
  - 8080
  - 9000
  - 3000

# Или короче для JSON-совместимости
ports: [8080, 9000, 3000]

# Объект (dictionary/map)
database:
  host: localhost
  port: 5432
  name: myapp

# Или JSON-стиль
database: {host: localhost, port: 5432}

# Список объектов
services:
  - name: web
    image: nginx
  - name: db
    image: postgres

# Вложенные структуры
app:
  web:
    port: 8080
    image: nginx
  db:
    port: 5432
    image: postgres

# Якоря и ссылки (для повторного использования)
defaults: &defaults
  restart: always
  logging:
    driver: json-file

web:
  <<: *defaults          # вставить defaults
  image: nginx

api:
  <<: *defaults          # вставить defaults
  image: myapi:latest
```

**Правила YAML:**
- ✅ Отступы обязательны (2 или 4 пробела, не табуляции)
- ✅ Дублирующиеся ключи запрещены
- ✅ Чувствительна к типам (true vs "true")
- ✅ YAML совместим с JSON (любой JSON — валидный YAML)

---

#### Структура docker-compose.yml

**Базовая структура:**
```yaml
version: '3.9'        # версия Docker Compose

services:             # основной блок
  web:
    image: nginx:latest
    ports:
      - "8080:80"
  
  db:
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: secret

networks:             # опционально
  default:
    driver: bridge

volumes:              # опционально
  db_data:
```

**Версии Docker Compose:**

| Версия | Docker | Примечание |
|--------|--------|-----------|
| 2.x | 1.10+ | Legacy, не рекомендуется |
| 3.0–3.8 | 1.13+ | Поддержка Swarm, базовый функционал |
| 3.9 | 20.10+ | Последняя v3, хороший баланс |
| 2.0+ (новая) | 20.10+ | Новая версия с дополнительными функциями |

**Рекомендуется использовать:** `3.9` или новую версию

---

#### Основные команды

**docker compose up:**
```bash
docker compose up                     # запустить в foreground
docker compose up -d                  # запустить в background
docker compose up --build             # пересобрать образы перед запуском
docker compose up service_name        # запустить конкретный сервис
```

**docker compose down:**
```bash
docker compose down                   # остановить и удалить контейнеры
docker compose down -v                # + удалить volumes
docker compose down --remove-orphans   # + удалить orphan контейнеры
```

**docker compose ps:**
```bash
docker compose ps                     # список контейнеров
docker compose ps -a                  # + остановленные контейнеры
```

**docker compose logs:**
```bash
docker compose logs                   # все логи
docker compose logs web               # логи конкретного сервиса
docker compose logs -f                # follow (streaming)
docker compose logs --tail 100        # последние 100 строк
```

**docker compose exec:**
```bash
docker compose exec web bash          # shell в контейнере
docker compose exec db psql -U user   # команда в контейнере
```

**docker compose stop/start:**
```bash
docker compose stop                   # остановить контейнеры
docker compose start                  # запустить остановленные
docker compose restart                # перезагрузить
```

**docker compose build:**
```bash
docker compose build                  # собрать образы
docker compose build --no-cache       # собрать без кэша
```

**docker compose config:**
```bash
docker compose config                 # показать итоговую конфигурацию
```

---

#### Простая конфигурация: Web + Database

**Пример: Nginx + PostgreSQL**

```yaml
version: '3.9'

services:
  web:
    image: nginx:latest
    container_name: my_web
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    networks:
      - app_net
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    container_name: my_db
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  app_net:
    driver: bridge

volumes:
  db_data:
```

**Использование:**
```bash
docker compose up -d                 # запустить
curl localhost:8080                  # доступ к nginx
docker compose logs db               # логи БД
docker compose exec db psql -U appuser -d myapp   # подключиться к БД
docker compose down -v               # остановить и очистить
```

---

#### Сетевое взаимодействие

**Автоматическая сеть:**
- Docker Compose автоматически создаёт сеть для сервисов
- Все сервисы подключены к этой сети
- Сервисы могут обращаться друг к другу по имени контейнера

**Service Discovery:**
```yaml
version: '3.9'

services:
  web:
    image: node:18
    # Может обращаться к app как: http://app:3000
    environment:
      API_URL: http://app:3000

  app:
    image: myapi:latest
    ports:
      - "3000:3000"
```

**Примеры обращения:**
```bash
# web контейнер может сделать:
curl http://app:3000/api/users        # DNS разрешится автоматически
ping app                               # работает (если не отключен ICMP)

# Это работает благодаря встроенному DNS сервером Docker
# (127.0.0.11:53 в контейнере)
```

**Явное определение сети:**
```yaml
version: '3.9'

services:
  web:
    image: nginx
    networks:
      - backend
      - frontend

  api:
    image: myapi
    networks:
      - backend

  cache:
    image: redis
    networks:
      - backend

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge

# web видит: api, cache (backend)
# web видит: nothing else (frontend изолирована)
# api видит: web (backend), cache (backend)
# cache видит: web, api (backend)
```

---

#### Ports vs Expose

**Ports (опубликовать порт на хосте):**
```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"         # хост:контейнер
      - "443:443"
      - "127.0.0.1:3000:3000"   # только localhost
```

**Expose (документирование, видно внутри сети):**
```yaml
services:
  api:
    image: myapi
    expose:
      - 3000              # видно из других контейнеров
    # НЕ видно с хоста!
```

**Различие:**
- `ports` — доступно снаружи (с хоста, из интернета)
- `expose` — видно только другим контейнерам в сети

---

#### Практические примеры

##### Пример 1: Node.js + MongoDB

```yaml
version: '3.9'

services:
  app:
    build: .                    # собрать из Dockerfile в текущей папке
    container_name: myapp
    ports:
      - "3000:3000"
    environment:
      MONGODB_URL: mongodb://db:27017/mydb
      NODE_ENV: development
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app                  # bind mount для development
      - /app/node_modules       # исключить node_modules
    networks:
      - app_net

  db:
    image: mongo:latest
    container_name: mydb
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - db_data:/data/db
    networks:
      - app_net
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  app_net:

volumes:
  db_data:
```

##### Пример 2: Python + PostgreSQL + Redis

```yaml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/myapp
      REDIS_URL: redis://cache:6379/0
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    volumes:
      - .:/app

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

volumes:
  db_data:
```

##### Пример 3: Full Stack (Frontend + Backend + DB)

```yaml
version: '3.9'

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3001:80"
    environment:
      REACT_APP_API_URL: http://localhost:8000
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  db_data:
```

---

#### Проблемы и решения

**Проблема: Контейнеры не видят друг друга**
```bash
# Решение: проверить сеть
docker compose ps              # все в одной сети?
docker compose inspect web     # просмотреть детали
docker compose exec web ping api    # тест connectivity
```

**Проблема: Port уже занят**
```bash
# Решение: использовать другой порт
# В docker-compose.yml:
ports:
  - "8081:80"        # вместо 8080

# Или через переменную:
ports:
  - "${WEB_PORT}:80"
# Запуск: WEB_PORT=8081 docker compose up
```

**Проблема: БД не готова когда приложение запускается**
```yaml
# Решение: healthcheck + depends_on condition
db:
  image: postgres
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U user"]
    interval: 10s

app:
  depends_on:
    db:
      condition: service_healthy    # ждать healthcheck
```
