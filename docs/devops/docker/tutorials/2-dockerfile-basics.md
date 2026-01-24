---
title: "2 Написание Dockerfile"
description: "Создаем первый образ для Node.js: работа с зависимостями, слои и безопасность."
---

Dockerfile — это сценарий создания образа. Давайте напишем его правильно.

## Задача: Собрать Node.js приложение

Создайте в пустой папке два файла:
1.  `Dockerfile`
2.  `index.js` (простой сервер)

### 1. Подготовка кода
Создайте `index.js`:
```javascript
const http = require('http');
const port = 3000;
const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.end('Hello Docker!\n');
});
server.listen(port, () => console.log(`Server running on port ${port}`));
```

И `package.json` (минимальный):
```json
{
  "name": "docker-test",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": { "start": "node index.js" },
  "dependencies": {}
}
```

### 2. Создание Dockerfile
Вставьте этот код в файл с именем `Dockerfile` (без расширения).

```dockerfile
# 1. Базовый образ
# Используем Alpine для минимизации размера (~50MB против 800MB у стандартного)
# Пиним версию (node:18, а не latest) для предсказуемости.
FROM node:18-alpine

# 2. Рабочая директория
# Все следующие команды будут выполняться внутри этой папки в контейнере.
WORKDIR /app

# 3. Кэширование зависимостей (Magic Step!)
# Сначала копируем ТОЛЬКО файлы описания зависимостей.
COPY package*.json ./

# Устанавливаем зависимости.
# `npm ci` — это более быстрый и надежный аналог `npm install` для CI/CD.
# Он удаляет node_modules и ставит все точно как в package-lock.json.
RUN npm ci --only=production

# 4. Копируем исходный код
# Мы делаем это ПОСЛЕ установки зависимостей.
# Если вы поменяете код в index.js, Docker пересоберет только этот слой,
# а тяжелый слой с node_modules возьмет из кэша.
COPY . .

# 5. Безопасность
# По умолчанию Docker работает от root. Это опасно.
# Переключаемся на встроенного пользователя 'node'.
USER node

# 6. Запуск
# Используем массив JSON (exec form), чтобы процесс получил PID 1
# и корректно принимал сигналы остановки (SIGTERM).
CMD ["node", "index.js"]
```

### 3. Сборка и Запуск

1.  **Сборка образа:**
    ```bash
    docker build -t my-node-app .
    ```
    *Точка в конце означает "искать Dockerfile в текущей папке".*

2.  **Запуск контейнера:**
    ```bash
    docker run -p 3000:3000 my-node-app
    ```

3.  **Проверка:**
    Откройте [http://localhost:3000](http://localhost:3000).

## Почему именно такой порядок? (Layers Caching)

Если вы напишете:
```dockerfile
COPY . .
RUN npm ci
```
То при **любом** изменении в `index.js` Docker сбросит кэш на строке `COPY` и будет заново переустанавливать все пакеты в `RUN`. Это долго.

Разделяя копирование `package.json` и остального кода, мы экономим минуты на сборке.
