---
title: "Написание первого Dockerfile"
description: "Практическое руководство по созданию Dockerfile для приложений на Node.js и Python."
---


В этом руководстве мы создадим эффективный образ для веб-приложения.

## Пример: Node.js приложение

Создайте файл с именем `Dockerfile` в корне проекта.

```dockerfile
# 1. Выбираем легкий базовый образ (Alpine Linux)
FROM node:18-alpine

# 2. Указываем рабочую папку
WORKDIR /app

# 3. Сначала копируем только файлы зависимостей!
# Это позволяет Docker кэшировать слой с node_modules, 
# если package.json не изменился.
COPY package*.json ./

# 4. Устанавливаем зависимости
RUN npm ci --only=production

# 5. Теперь копируем исходный код
# Этот слой будет меняться часто, но предыдущие шаги возьмутся из кэша.
COPY . .

# 6. Документируем порт
EXPOSE 3000

# 7. Запускаем от непривилегированного пользователя (Security)
USER node

# 8. Команда запуска
CMD ["node", "index.js"]
```

### Сборка и запуск

```bash
# Сборка образа с тегом my-node-app
docker build -t my-node-app .

# Запуск
docker run -p 3000:3000 my-node-app
```

## Пример: Python приложение

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Копируем зависимости
COPY requirements.txt .

# Устанавливаем без кэша pip (экономим место)
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Создаем пользователя (в python образах его часто нет по умолчанию)
RUN useradd -m appuser
USER appuser

CMD ["python", "app.py"]
```

## Советы по оптимизации слоев

Порядок команд важен для кэширования. Docker проверяет слои сверху вниз. Как только он находит изменение (например, в `COPY . .`), **все** последующие слои пересобираются заново.

**Правило:** Размещайте редко изменяемые инструкции (установка системных библиотек, `npm install`) **выше**, а часто изменяемые (исходный код) — **ниже**.

## Связанные материалы

- [[devops/docker/reference/dockerfile-instructions|Справочник инструкций]]
- [[devops/docker/explanation/images-and-layers|Понимание слоев]]
