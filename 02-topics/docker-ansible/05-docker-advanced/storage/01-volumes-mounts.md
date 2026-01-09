### Docker Storage: Volumes и Bind Mounts

Хранение и сохранение данных в Docker.

---

#### Зачем нужны Volumes

**Проблемы без Volumes:**
- Данные в контейнере удаляются при удалении контейнера
- Невозможно достучаться к данным извне
- Сложно обновлять приложение без потери данных

**Решения Volumes:**
- **Персистентное хранение** — данные сохраняются между перезапусками
- **Экспортирование логов** — сбор логов внешними системами
- **Передача конфигураций** — обновление настроек без пересборки образа
- **Шейринг данных** — несколько контейнеров работают с одними данными

---

#### Типы хранилища в Docker

| Тип | Управление | Использование | Персистентность |
|-----|-----------|---------------|-----------------|
| **Named Volumes** | Docker | Основной метод, БД | ✅ Да |
| **Anonymous Volumes** | Docker | Временные данные | ✅ Да |
| **Bind Mounts** | Хост | Development, конфиги | ✅ Да |
| **Tmpfs** | Память | Secrets, временное | ❌ Нет |

---

#### Named Volumes (рекомендуется)

**Что это:**
- Специальная папка внутри Docker (обычно `/var/lib/docker/volumes/`)
- Управляется Docker, не нужно думать о путях
- Синхронизирована с папкой в контейнере

**Создание:**
```bash
docker volume create mydata                   # создать volume
docker volume create --driver local mydata    # явно с драйвером
```

**Просмотр:**
```bash
docker volume ls                              # список всех volumes
docker volume inspect mydata                  # информация о volume
# Вывод содержит Mountpoint (путь на хосте)
```

**Запуск контейнера с volume:**
```bash
# Способ 1: -v флаг (простой)
docker run -d --name myapp -v mydata:/app/data nginx

# Способ 2: --mount (явный, рекомендуется)
docker run -d --name myapp --mount source=mydata,target=/app/data nginx

# Способ 3: read-only
docker run -d --name myapp -v mydata:/app/data:ro nginx
```

**Параметры:**
- `source` / первый аргумент `-v` — имя volume на хосте
- `target` / второй аргумент `-v` — путь в контейнере
- `:ro` — read-only режим
- `:rw` — read-write (по умолчанию)

**Sharing между контейнерами:**
```bash
# Контейнер 1 (БД)
docker run -d --name db -v mydata:/data postgres

# Контейнер 2 (приложение)
docker run -d --name app -v mydata:/app-data myapp
# Оба контейнера работают с одними данными
```

**Backup volume:**
```bash
# Сохранить volume в архив
docker run --rm -v mydata:/data -v $(pwd):/backup \
  ubuntu tar czf /backup/mydata.tar.gz /data

# Restore volume из архива
docker run --rm -v mydata:/data -v $(pwd):/backup \
  ubuntu tar xzf /backup/mydata.tar.gz -C /
```

**Удаление:**
```bash
docker volume rm mydata                       # удалить volume
docker volume prune                           # удалить все неиспользуемые
docker volume prune -a                        # все volumes
```

---

#### Bind Mounts

**Что это:**
- Прямое биндирование директории хоста с контейнером
- Полный контроль над путём на хосте
- Для development и конфигураций

**Когда использовать:**
- **Development** — синхронизация кода между хостом и контейнером
- **Конфигурации** — обновление config файлов без пересборки
- **Логи** — сбор логов в определённую папку

**Синтаксис:**
```bash
# Способ 1: -v (простой)
docker run -d -v /home/user/code:/app myapp

# Способ 2: --mount (явный)
docker run -d --mount type=bind,source=/home/user/code,target=/app myapp

# Способ 3: read-only
docker run -d -v /home/user/config:/etc/myapp:ro myapp
```

**Практические примеры:**

**Development: Node.js приложение**
```bash
# Код на хосте синхронизируется с контейнером
docker run -d \
  --name dev-app \
  -v $(pwd)/src:/app/src \
  -v /app/node_modules \
  -p 3000:3000 \
  node:18 npm start
```

**Production: конфигурация**
```bash
# Config на хосте (read-only в контейнере)
docker run -d \
  --name nginx-prod \
  -v /etc/myapp/nginx.conf:/etc/nginx/nginx.conf:ro \
  -p 80:80 \
  nginx
```

**Логи:**
```bash
# Логи из контейнера на хост
docker run -d \
  --name app \
  -v /var/log/myapp:/app/logs \
  myapp:latest
```

**Важно:**
- Если папка на хосте не существует, она будет создана
- Содержимое может перезаписать содержимое в контейнере
- Bind mounts видят изменения в реальном времени

**Разница от Named Volumes:**

| | Named Volumes | Bind Mounts |
|---|---|---|
| Управление | Docker | Хост |
| Путь на хосте | Docker выбирает | Ты выбираешь |
| Использование | БД, persistent data | Development, configs |
| Production | ✅ Лучше | ⚠️ Нужна осторожность |

---

#### Anonymous Volumes

**Что это:**
- Unnamed volumes, создаются автоматически
- Управляются Docker
- Обычно используются для VOLUME инструкций в Dockerfile

**Пример:**
```bash
docker run -d -v /app/data myapp
# Docker создает случайный volume для /app/data
# После удаления контейнера остаётся "dangling" volume
```

**Очистка:**
```bash
docker volume prune                    # удалить dangling volumes
```

---

#### Tmpfs Mounts (временное хранилище)

**Что это:**
- Данные хранятся в памяти (ОЗУ), не на диске
- Удаляются при остановке контейнера
- Для секретных данных и высокопроизводительных операций

**Когда использовать:**
- **Secrets** — пароли, API ключи (не сохранять на диск)
- **Session data** — временные данные сессии
- **Cache** — высокоскоростной кэш

**Синтаксис:**
```bash
# Способ 1: --tmpfs
docker run -d --tmpfs /app/secrets myapp

# Способ 2: --mount
docker run -d --mount type=tmpfs,destination=/app/secrets myapp

# С опциями размера
docker run -d --tmpfs /app/secrets:size=1g,mode=1777 myapp
```

**Пример: Redis кэш**
```bash
# Redis в памяти для сессий
docker run -d \
  --name redis-cache \
  --tmpfs /data \
  -p 6379:6379 \
  redis:latest
```

**Важно:**
- Данные НЕ сохраняются
- Не работает с `docker cp`
- Не работает с volumes в Swarm

---

#### VOLUME инструкция в Dockerfile

**Что это:**
- Объявление mount point'ов в образе
- Не создаёт volume, только документирует
- Автоматически создаёт anonymous volumes

**Синтаксис:**
```dockerfile
FROM postgres:latest

# Объявляем mount point для данных
VOLUME /var/lib/postgresql/data
```

**Поведение:**
```bash
# При запуске контейнера
docker run -d postgres
# Автоматически создаётся anonymous volume для /var/lib/postgresql/data
```

**Best practice:**
```dockerfile
# Правильно: объявляем все mount points
FROM node:18
WORKDIR /app
VOLUME /app/data
VOLUME /app/logs

CMD ["node", "index.js"]
```

---

#### docker cp: Копирование файлов

**Когда использовать:**
- Когда volume не подключён
- Для временного копирования файлов
- Для извлечения логов или конфигов

**Синтаксис:**
```bash
# Копировать из хоста в контейнер
docker cp /path/on/host container_name:/path/in/container

# Копировать из контейнера на хост
docker cp container_name:/path/in/container /path/on/host

# Копировать папку
docker cp /home/user/folder container_name:/app/folder
```

**Примеры:**

**Скопировать конфиг в контейнер:**
```bash
docker cp app.conf mycontainer:/app/config/app.conf
```

**Извлечь логи из контейнера:**
```bash
docker cp mycontainer:/app/logs/app.log ./app.log
```

**Копировать папку:**
```bash
docker cp ./data mycontainer:/app/data
```

**Важно:**
- Контейнер должен быть запущен (или остановлен для read-only операций)
- Не создаёт volume, только копирует файлы
- Медленнее чем volumes для частых операций

---

#### Практические примеры

##### PostgreSQL с Named Volume

```bash
# Создать volume
docker volume create pgdata

# Запустить PostgreSQL
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:latest

# Данные сохранятся даже если удалить контейнер
docker rm postgres

# Запустить новый контейнер с теми же данными
docker run -d \
  --name postgres-new \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:latest
```

##### Development: Node.js + MongoDB

```bash
# Создать network
docker network create dev-net

# MongoDB с volume
docker run -d \
  --name mongo \
  --network dev-net \
  -v mongodata:/data/db \
  mongo:latest

# Node приложение с bind mount
docker run -d \
  --name node-app \
  --network dev-net \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  node:18 npm start

# Изменения в src/ видны в контейнере в реальном времени
```

##### Backup/Restore MongoDB

```bash
# Backup volume
docker run --rm \
  -v mongodata:/data \
  -v $(pwd)/backups:/backups \
  mongo:latest \
  mongodump --out /backups/dump

# Restore volume
docker run --rm \
  -v mongodata:/data \
  -v $(pwd)/backups:/backups \
  mongo:latest \
  mongorestore /backups/dump
```

##### Production: конфигурация + логи

```bash
docker run -d \
  --name production-app \
  -v /etc/myapp/config.yml:/app/config.yml:ro \
  -v /var/log/myapp:/app/logs \
  -p 3000:3000 \
  myapp:latest

# Логи доступны на хосте
cat /var/log/myapp/app.log
```

---

#### Best Practices

✅ **Используй named volumes** для production и БД  
✅ **Используй bind mounts** для development и configs  
✅ **Используй tmpfs** для secrets и cache  
✅ **Объявляй VOLUME** в Dockerfile для mount points  
✅ **Регулярно очищай** dangling volumes (`docker volume prune`)  
✅ **Делай backup** важных volumes  
✅ **Используй read-only** для config файлов когда возможно  
✅ **Избегай** больших файлов в volumes для network shares  
