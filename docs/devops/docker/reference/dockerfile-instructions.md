---
title: "Справочник инструкций Dockerfile"
description: "Полный список инструкций Dockerfile: FROM, RUN, CMD, COPY, ADD, ENTRYPOINT и другие с примерами использования."
---


Dockerfile — это сценарий сборки образа. Каждая инструкция создает новый слой (за исключением метаданных).

## Основные инструкции

### FROM
Определяет базовый образ. Должна быть первой инструкцией.
```dockerfile
FROM node:18-alpine
# Или с псевдонимом для multi-stage сборки
FROM golang:1.21 AS builder
```

### WORKDIR
Устанавливает рабочую директорию. Если директории нет, она будет создана.
```dockerfile
WORKDIR /app
# Следующие команды (RUN, CMD, COPY) будут выполняться в /app
```

### COPY
Копирует файлы из контекста сборки (ваш проект) в образ.
```dockerfile
# COPY <src> <dest>
COPY package.json .
COPY src/ /app/src/
```

### RUN
Выполняет команду во время сборки образа и фиксирует результат в новом слое.
```dockerfile
# Shell form (запускается через /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (предпочтительно, без shell)
RUN ["npm", "install"]
```

### CMD
Указывает команду, которая выполнится **при запуске контейнера**.
Может быть переопределена аргументами `docker run`.
```dockerfile
# Exec form (рекомендуется)
CMD ["node", "app.js"]
# Shell form
CMD node app.js
```
*В Dockerfile может быть только одна инструкция CMD (последняя).*

### ENTRYPOINT
Настраивает контейнер как исполняемый файл. Аргументы `docker run` добавляются к ENTRYPOINT.
```dockerfile
ENTRYPOINT ["npm"]
CMD ["start"]
# Результат: выполняется `npm start`.
# Если запустить `docker run myapp test`, выполнится `npm test`.
```

### ENV
Устанавливает постоянные переменные окружения.
```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

### EXPOSE
Информирует Docker (и человека), что контейнер слушает указанный порт. Не пробрасывает порт автоматически.
```dockerfile
EXPOSE 8080
```

### USER
Переключает пользователя для последующих команд.
```dockerfile
USER node
```

### ARG
Переменные, доступные **только во время сборки**.
```dockerfile
ARG VERSION=latest
FROM alpine:${VERSION}
```

## .dockerignore

Файл `.dockerignore` в корне проекта исключает файлы из контекста сборки, ускоряя процесс.
```text
node_modules
.git
.env
*.log
```

## Связанные материалы

- [[devops/docker/tutorials/dockerfile-basics|Написание первого Dockerfile (Туториал)]]
- [[devops/docker/tutorials/optimize-images|Оптимизация размера и кэширования]]
