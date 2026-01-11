### Dockerfile: основы

Написание Dockerfile для создания собственных Docker образов.

---

#### Что такое Dockerfile

**Определение:**
- Текстовый файл с инструкциями для сборки Docker образа
- Каждая инструкция создаёт новый слой
- Формат: `ИНСТРУКЦИЯ аргументы`

**Структура:**
```dockerfile
# комментарий
FROM базовый_образ
RUN команда
COPY файл /путь
...
```

---

#### Сборка образа

**Команда docker build:**
```bash
docker build -t myapp:1.0 .                       # с тегом
docker build -t myapp:latest -t myapp:1.0 .       # несколько тегов
docker build -f Dockerfile.prod -t app:prod .     # другой файл
docker build --build-arg NODE_ENV=production .    # с аргументами
```

**Параметры:**
- `-t, --tag` — имя и тег образа
- `-f, --file` — путь к Dockerfile (по умолчанию `./Dockerfile`)
- `--build-arg` — переменные для сборки
- `.` (точка) — контекст сборки (обычно текущая директория)

**Контекст сборки:**
- Файлы, которые Docker может копировать в образ
- Отправляется Docker daemon'у
- Большой контекст = медленнее сборка

---

#### .dockerignore файл

**Исключение файлов из контекста:**
```dockerfile
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.DS_Store
*.log
coverage/
dist/
build/
```

**Результат:**
- Меньше размер контекста
- Быстрее сборка
- Безопаснее (исключаются .env файлы)

---

#### Основные инструкции Dockerfile

##### FROM (обязательная, первая)

```dockerfile
FROM ubuntu:22.04                    # базовый образ
FROM node:18-alpine                  # Node.js на Alpine Linux
FROM python:3.11-slim                # Python на Debian
FROM scratch                          # пустой образ (для бинарников)
```

**Обязательно:**
- Первая инструкция в Dockerfile
- Устанавливает базовый слой для всех следующих команд

---

##### WORKDIR (рабочая директория)

```dockerfile
WORKDIR /app                         # создаёт /app и переходит туда
WORKDIR backend                      # относительный путь от предыдущего
```

**Что делает:**
- Создаёт директорию если её нет
- Устанавливает рабочую директорию для последующих команд
- Используется в RUN, COPY, CMD, и т.д.

---

##### COPY / ADD (добавление файлов)

**COPY (рекомендуется):**
```dockerfile
COPY package.json .                  # копирует в WORKDIR
COPY . .                              # копирует всё из контекста
COPY app.js src/                      # копирует в src/ директорию
```

**ADD (дополнительные возможности):**
```dockerfile
ADD https://example.com/file.tar.gz . # загрузить из URL
ADD file.tar.gz .                     # автоматически распаковать
```

**Разница:**
- COPY просто копирует файлы
- ADD может загружать с URL и распаковывать архивы
- COPY предпочтительнее для ясности

---

##### RUN (выполнение команд)

```dockerfile
RUN apt-get update && apt-get install -y nginx     # команда shell
RUN ["npm", "install"]                             # exec форма
RUN npm install && npm run build                    # несколько команд
```

**Параметры:**
- По умолчанию `/bin/sh -c` (shell форма)
- Exec форма не использует shell: `["executable", "param1", "param2"]`

**Best practice:**
- Группируй RUN команды для уменьшения слоёв
- Очищай кэш пакетов: `apt-get clean && rm -rf /var/lib/apt/lists/*`

```dockerfile
# Плохо (много слоёв)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# Хорошо (один слой)
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*
```

---

##### ENV (переменные окружения)

```dockerfile
ENV NODE_ENV production
ENV PORT 3000
ENV DATABASE_URL postgresql://localhost/mydb
```

**Использование:**
- Переменные доступны в контейнере и при сборке
- Можешь переопределить при запуске: `docker run -e NODE_ENV=dev`

---

##### EXPOSE (документирование портов)

```dockerfile
EXPOSE 3000
EXPOSE 3000 5432
```

**Важно:**
- Это просто **документация**, не открывает порт
- Для открытия используй `docker run -p 3000:3000`
- Помогает разработчикам понять какой порт нужен

---

##### USER (смена пользователя)

```dockerfile
USER nobody                          # неприватилегированный пользователь
USER appuser                         # создай пользователя перед этим
```

**Best practice:**
- Запускай приложения от неприватилегированного пользователя
- Избегай запуска от root

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
```

---

##### LABEL (метаинформация)

```dockerfile
LABEL version="1.0"
LABEL maintainer="dev@example.com"
LABEL description="My awesome app"
LABEL org.opencontainers.image.source=https://github.com/user/repo
```

---

#### Запуск контейнера

##### CMD (команда по умолчанию)

```dockerfile
CMD ["node", "app.js"]               # exec форма (рекомендуется)
CMD node app.js                       # shell форма
```

**Особенности:**
- Может быть переопределена при запуске
- `docker run myapp npm test` переопределит CMD
- Может быть несколько CMD, но только последняя выполняется

---

##### ENTRYPOINT (точка входа)

```dockerfile
ENTRYPOINT ["node", "app.js"]        # exec форма
ENTRYPOINT node app.js                # shell форма
```

**Разница от CMD:**
- ENTRYPOINT обычно не переопределяется
- `docker run myapp --help` передаст `--help` как аргумент

---

##### ENTRYPOINT vs CMD

**Комбинирование:**
```dockerfile
ENTRYPOINT ["npm"]
CMD ["start"]
```

**Результаты:**
- `docker run myapp` выполнит: `npm start`
- `docker run myapp test` выполнит: `npm test`

**Таблица:**

| ENTRYPOINT | CMD | Результат |
|-----------|-----|-----------|
| `node` | `app.js` | `node app.js` |
| `node` | — | `node` |
| — | `npm start` | `npm start` |
| `[node]` | `[app.js]` | `node app.js` |

---

#### ARG (аргументы сборки)

```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

ARG BUILD_DATE
LABEL build.date=$BUILD_DATE
```

**При сборке:**
```bash
docker build --build-arg NODE_VERSION=20 .
docker build --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') .
```

**Разница от ENV:**
- ARG доступен только во время сборки
- ENV доступен в контейнере

---

#### Практические примеры

##### Node.js приложение

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

USER node

CMD ["node", "index.js"]
```

##### Python приложение

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

##### Go приложение

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN go build -o app .

FROM alpine:latest
COPY --from=builder /app/app .
CMD ["./app"]
```

---

#### Порядок инструкций (для оптимизации)

**Хороший порядок:**
```dockerfile
FROM node:18-alpine              # 1. Базовый образ

WORKDIR /app                     # 2. Рабочая директория

ENV NODE_ENV production          # 3. Переменные (редко меняются)

COPY package*.json ./            # 4. Зависимости (редко меняются)
RUN npm ci --only=production

COPY . .                          # 5. Код (часто меняется)

EXPOSE 3000                       # 6. Документация

USER node                         # 7. Security

CMD ["node", "index.js"]          # 8. Запуск
```

**Почему так:**
- Изменяющиеся вещи должны быть в конце
- Слои кэшируются отдельно
- При изменении кода не переустанавливай зависимости

---
