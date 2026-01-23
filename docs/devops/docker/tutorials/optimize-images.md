---
title: "Оптимизация Docker образов"
description: "Практическое руководство по уменьшению размера образа и ускорению сборки: выбор базового образа, многоэтапная сборка (multi-stage) и правильная работа с кэшем."
---


Маленькие образы быстрее загружаются, занимают меньше места и безопаснее (меньше поверхность атаки). В этом руководстве мы уменьшим размер Node.js образа с 1.2 ГБ до 180 МБ.

## 1. Выбор базового образа

Первый шаг к оптимизации — использование специализированных тегов.

* **`ubuntu`/`debian`**: Полноценные ОС. Тяжелые (~100 МБ+), но удобные для отладки.
* **`slim`** (напр. `python:3.11-slim`): Урезаная версия Debian. Оптимальный баланс (~40-50 МБ).
* **`alpine`** (напр. `node:18-alpine`): Экстремально легкий Linux (~5 МБ). Использует `musl` вместо `glibc` (могут быть нюансы совместимости), но идеален для большинства сервисов.

**Сравнение размеров:**
| Образ | Размер |
|---|---|
| `python:3.11` | 1 ГБ |
| `python:3.11-slim` | 130 МБ |
| `python:3.11-alpine` | 50 МБ |

## 2. Эффективное кэширование (Layer Caching)

Docker кэширует каждый слой. Если слой не изменился, Docker использует кэш.

**Плохо:**
```dockerfile
COPY . .
RUN npm install
# Если вы поменяете хоть одну букву в коде,
# слой COPY . . изменится, и npm install запустится заново.
```

**Хорошо:**
```dockerfile
# Сначала копируем только файлы зависимостей
COPY package*.json ./
# Устанавливаем зависимости (этот слой закешируется надолго)
RUN npm ci --only=production

# Теперь копируем код
COPY . .
# При изменении кода пересобирается только этот слой, а npm install берется из кэша.
```

## 3. Multi-stage builds (Многоэтапная сборка)

Для компилируемых языков (Go, Java) или фронтенда (React build) нам нужны инструменты сборки (компилятор, devDependencies), которые не нужны в продакшене.

**Пример (Node.js):**
```dockerfile
# Этап 1: Сборка (Builder)
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
# Устанавливаем ВСЕ зависимости (включая dev)
RUN npm ci
COPY . .
# Собираем проект (создается папка dist/)
RUN npm run build

# Этап 2: Финальный образ (Production)
FROM node:18-alpine
WORKDIR /app
# Устанавливаем только prod-зависимости
COPY package*.json ./
RUN npm ci --only=production
# Копируем из этапа builder только готовую папку dist
COPY --from=builder /app/dist ./dist

CMD ["node", "dist/index.js"]
```
*Результат: в финальном образе нет исходников TypeScript, нет Webpack, нет лишних node_modules.*

## 4. Очистка кэшей пакетного менеджера

Если вы устанавливаете системные утилиты, очищайте кэш в **той же** инструкции `RUN`.

```dockerfile
# Ubuntu/Debian
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Alpine (флаг --no-cache делает это автоматически)
RUN apk add --no-cache curl
```

## Инструменты анализа

Утилита **dive** позволяет заглянуть внутрь слоев и найти "мусор".

```bash
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive my-app:latest
```
Она покажет эффективность (Efficiency Score) и файлы, которые были удалены в верхних слоях, но все еще занимают место в нижних.

## Связанные материалы

- [[2-images-and-layers|Как устроены слои]]
- [[devops/docker/reference/dockerfile-instructions|Справочник Dockerfile]]
