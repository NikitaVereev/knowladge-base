### Жизненный цикл контейнера

Управление контейнерами от создания до удаления.

---

#### Основные команды

**Справка:**
```bash
docker help                 # все доступные команды
docker <command> --help     # справка по команде
```

**Сокращение команд:**
```bash
docker run          # вместо docker container run
docker start        # вместо docker container start
docker stop         # вместо docker container stop
```

---

#### Создание и запуск

**Способ 1: create + start (два этапа)**
```bash
docker create --name myapp nginx:latest      # создание
docker start myapp                           # запуск
```

**Способ 2: run (один этап)**
```bash
docker run --name myapp nginx:latest         # создание + запуск
docker run --name myapp -d nginx:latest      # в фоне (-d, detach)
docker run --name myapp -it ubuntu bash      # интерактивно (-it)
```

**Результат:** Контейнер в состоянии `Running`.

---

#### Остановка контейнера

**Graceful остановка (с сигналом SIGTERM, 10 сек на завершение):**
```bash
docker stop myapp                            
docker stop --time=20 myapp                  # кастомное время
```

**Принудительная остановка (SIGKILL, немедленно):**
```bash
docker kill myapp
```

**Разница:**
- `stop` дает приложению время завершить работу (write files, close connections)
- `kill` останавливает немедленно (может потерять данные)

**Результат:** Контейнер в состоянии `Stopped` (Exit code 0 vs 137 для kill).

---

#### Перезапуск

**Остановить и запустить:**
```bash
docker restart myapp                         
docker restart --time=5 myapp                # кастомное время на остановку
```

**Эквивалент:**
```bash
docker stop myapp && docker start myapp
```

---

#### Пауза и возобновление

**Заморозить процесс:**
```bash
docker pause myapp                           
```

**Разморозить процесс:**
```bash
docker unpause myapp                         
```

**Разница от stop:**
- `pause` замораживает процесс, но контейнер остается в памяти
- `stop` останавливает контейнер полностью
- `pause` быстрее для временного приостановления

---

#### Удаление

**Удалить остановленный контейнер:**
```bash
docker rm myapp                              
```

**Удалить принудительно (даже если работает):**
```bash
docker rm -f myapp                           
```

**Удалить все остановленные контейнеры:**
```bash
docker container prune                       # явно
```

**Важно:** После удаления все данные контейнера теряются (кроме volumes).

---

#### Просмотр контейнеров

**Только активные (Running):**
```bash
docker ps
docker ps -q                                 # только ID
docker ps --format "{{.Names}} {{.Status}}"  # кастомный формат
```

**Все контейнеры (включая Stopped, Exited):**
```bash
docker ps -a
docker ps -a -q
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"
```

**Фильтрация:**
```bash
docker ps -a --filter "status=exited"
docker ps -a --filter "name=myapp"
docker ps -a --filter "ancestor=nginx"       # все контейнеры от образа nginx
```

---

#### Дополнительные команды

**Переименование:**
```bash
docker rename old_name new_name
```

**Информация о контейнере:**
```bash
docker inspect myapp                         # JSON со всеми деталями
docker inspect --format '{{.State.Status}}' myapp   # конкретное поле
```

**История контейнеров:**
```bash
docker ps -a --format "{{.ID}}\t{{.Image}}\t{{.CreatedAt}}"
```

**Очистка неиспользуемых ресурсов:**
```bash
docker container prune                       # остановленные контейнеры
docker image prune                           # неиспользуемые образы
docker volume prune                          # неиспользуемые volumes
docker system prune -a                       # всё сразу
```

---

#### Stateful контейнеры

**Важно:** Контейнер сохраняет свое состояние между `stop/start`.

```bash
# 1. Создаем контейнер
docker run -d --name mongodb mongo:latest

# 2. Добавляем файл
docker exec mongodb touch /myfile.txt

# 3. Останавливаем
docker stop mongodb

# 4. Запускаем снова
docker start mongodb

# 5. Файл сохранился!
docker exec mongodb ls /myfile.txt   # ✅ файл есть
```

**Но если удалить:**
```bash
docker rm mongodb  # файл УДАЛИТСЯ
```

**Для сохранения данных используй volumes (следующие разделы).**

---

#### Практический пример

```bash
# 1. Создание и запуск
docker run -d --name db -p 27017:27017 mongo:latest

# 2. Проверка статуса
docker ps
docker ps -a

# 3. Просмотр информации
docker inspect db | grep -A 5 "State"

# 4. Выполнение команды в контейнере
docker exec db mongosh --version

# 5. Остановка и проверка
docker stop db
docker ps -a    # STATUS: Exited

# 6. Запуск снова
docker start db
docker ps       # STATUS: Up

# 7. Переименование
docker rename db database

# 8. Удаление
docker rm -f database
docker ps -a    # контейнера нет
```

---

#### Диаграмма состояний и команд

```
          ┌─────────────┐
          │   Created   │
          └──────┬──────┘
                 │
            docker start
                 │
    ┌────────────▼──────────────┐
    │                           │
docker pause              docker stop
    │                           │
┌───▼──────┐            ┌───────▼────┐
│  Paused   │            │   Stopped  │
└───┬──────┘            └────┬───────┘
    │                         │
docker unpause          docker start
    │                         │
    └────────────┬────────────┘
                 │
           ┌─────▼─────┐
           │  Running   │
           └─────┬─────┘
                 │
            docker kill
              или stop
                 │
            docker rm
                 │
            Removed
```

---
