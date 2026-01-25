---
title: 2 Инструкции Dockerfile
description: Справочник по синтаксису Dockerfile. Различия COPY/ADD, CMD/ENTRYPOINT и ARG/ENV.
---

## Основные инструкции

| Инструкция | Описание | Пример |
| :--- | :--- | :--- |
| **FROM** | Базовый образ. Первая инструкция. | `FROM node:20-alpine` |
| **WORKDIR** | Задает рабочую директорию для след. команд. | `WORKDIR /app` |
| **COPY** | Копирует файлы с хоста в образ. | `COPY . .` |
| **ADD** | Как COPY, но умеет распаковывать tar и качать URL. | `ADD https://.../file.tar.gz /src` |
| **RUN** | Выполняет команду *при сборке* (создает слой). | `RUN npm install` |
| **CMD** | Команда по умолчанию *при запуске*. Можно переопределить. | `CMD ["node", "app.js"]` |
| **ENTRYPOINT** | Основной процесс. Аргументы CMD летят к нему. | `ENTRYPOINT ["/app/start.sh"]` |
| **ENV** | Переменная окружения (доступна в build и runtime). | `ENV PORT=3000` |
| **ARG** | Переменная сборки (доступна ТОЛЬКО в build). | `ARG VERSION=1.0` |
| **EXPOSE** | Документирует порт (не публикует его!). | `EXPOSE 80` |
| **VOLUME** | Объявляет точку монтирования (создает анонимный том). | `VOLUME /var/lib/mysql` |
| **USER** | Переключает пользователя (Security). | `USER node` |
| **HEALTHCHECK** | Проверка здоровья контейнера. | `HEALTHCHECK CMD curl -f ...` |
| **ONBUILD** | Триггер для дочерних образов (использовать с осторожностью). | `ONBUILD COPY . .` |

## Сравнительные таблицы

### COPY vs ADD
| Фича | COPY | ADD |
| :--- | :---: | :---: |
| Копирование локальных файлов | ✅ | ✅ |
| Скачивание по URL | ❌ | ✅ |
| Распаковка архивов (tar, gzip) | ❌ | ✅ |
| **Рекомендация** | **Всегда** | Только если нужна распаковка |

### CMD vs ENTRYPOINT
| Dockerfile | Команда `docker run` | Результат (PID 1) |
| :--- | :--- | :--- |
| `CMD ["echo", "hi"]` | (пусто) | `echo hi` |
| `CMD ["echo", "hi"]` | `bash` | `bash` (CMD игнорируется) |
| `ENTRYPOINT ["echo"]` | `hello` | `echo hello` (аргументы склеились) |
| `ENTRYPOINT ["echo"]` `CMD ["hi"]` | (пусто) | `echo hi` (CMD стал дефолтным аргументом) |
| `ENTRYPOINT ["echo"]` `CMD ["hi"]` | `bye` | `echo bye` (аргумент переопределил CMD) |

### Shell form vs Exec form
| Форма | Синтаксис | PID 1 | Сигналы (SIGTERM) |
| :--- | :--- | :--- | :--- |
| **Exec (JSON)** | `CMD ["node", "app.js"]` | `node` | ✅ Приходят |
| **Shell** | `CMD node app.js` | `/bin/sh -c ...` | ❌ Не приходят |
| **Рекомендация** | **Всегда Exec** | | |
