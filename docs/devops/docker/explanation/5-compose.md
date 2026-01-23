---
title: "5 Docker Compose"
description: "Полное руководство: от базового синтаксиса до продвинутых функций Watch, Profiles и Override для production-окружений."
---

Docker Compose — это инструмент для определения и запуска многоконтейнерных приложений. С его помощью вы описываете всю архитектуру системы (база данных, бэкенд, кеш) в одном файле `compose.yaml` (ранее `docker-compose.yml`), превращая инфраструктуру в код (IaC).

## Базовая структура и Синтаксис

Compose использует YAML. Главное правило: отступы (2 или 4 пробела) определяют вложенность.

### Пример `compose.yaml`

```yaml
name: my-project  # Имя проекта (вместо папки по умолчанию)

services:       # Секция описания контейнеров
  webapp:       # Имя сервиса (становится DNS-именем внутри сети)
    image: nginx:alpine
    ports:
      - "8080:80"  # Проброс порта Host:Container
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db      # Ждать запуска базы данных
    networks:
      - backend

  db:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app_db
    volumes:
      - db-data:/var/lib/postgresql/data # Именованный volume для сохранения данных
    networks:
      - backend

volumes:        # Регистрация томов
  db-data:

networks:       # Регистрация сетей
  backend:
```

### Сетевая магия
По умолчанию Compose создает общую сеть.
*   **Service Discovery:** Контейнеры видят друг друга по именам сервисов. Webapp может подключиться к базе по хосту `db` (например, `postgres://db:5432`).
*   **Изоляция:** Эта сеть изолирована от других проектов на вашем компьютере.

---

## Продвинутые возможности

Для сложных приложений и разделения dev/prod окружений используются Profiles, Overrides и функции вроде Watch.

### 1. Профили (Profiles)
Позволяют включать/выключать группы сервисов. Удобно для монорепозиториев (где есть `frontend`, `backend`, `worker`, `monitoring`), когда вам не нужно запускать всё сразу.

```yaml
services:
  web:
    image: my-app
    # Запускается всегда

  worker:
    image: my-worker
    profiles: ["workers"] # Запустится только если явно попросить

  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]
```

**Запуск:**
```bash
docker compose up                   # Только web
docker compose --profile workers up # web + worker
```

### 2. Режим Watch (Hot Reload 2.0)
Функция, заменяющая костыли с bind-mounts для разработки. Позволяет синхронизировать файлы или пересобирать контейнер на лету.

```yaml
services:
  web:
    build: .
    develop:
      watch:
        - action: sync          # Просто копировать файл в контейнер
          path: ./web           # Откуда (хост)
          target: /app/web      # Куда (контейнер)
          ignore:
            - node_modules/
        
        - action: rebuild       # Пересобрать имидж, если изменился package.json
          path: ./package.json
```
**Запуск:** `docker compose up --watch`

### 3. Переопределение (Override) и Наследование
Compose автоматически объединяет `compose.yaml` и `compose.override.yml`. Это стандарт для разделения конфигов.

*   **`compose.yaml`**: Базовая структура (имена образов, связи).
*   **`compose.override.yml`**: Настройки для локальной разработки (порты, volumes с кодом, режим отладки). **Не добавляется в Git.**
*   **`compose.prod.yml`**: Настройки для продакшена (политики рестарта, лимиты ресурсов).

**DRY: YAML Якоря (Anchors)**
Если у вас 5 микросервисов с одинаковыми переменными окружения, используйте якоря, чтобы не дублировать код.

```yaml
x-logging: &default-logging  # Определяем шаблон (Extension field)
  driver: "json-file"
  options:
    max-size: "10m"

services:
  api:
    image: my-api
    logging: *default-logging # Применяем шаблон
  
  worker:
    image: my-worker
    logging: *default-logging # Применяем шаблон
```

## Основные команды

| Команда | Описание |
| :--- | :--- |
| `docker compose up -d` | Запустить всё в фоне (Detached) |
| `docker compose down` | Остановить и **удалить** контейнеры и сеть |
| `docker compose ps` | Показать статус сервисов проекта |
| `docker compose logs -f` | Читать логи всех сервисов в реальном времени |
| `docker compose exec web bash` | Зайти внутрь контейнера `web` |
