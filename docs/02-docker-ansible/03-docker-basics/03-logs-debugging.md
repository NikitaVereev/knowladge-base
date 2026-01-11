### Логи и отладка контейнеров

Мониторинг, логирование и выполнение команд в контейнерах.

---

#### Статистика контейнера

**Мониторинг ресурсов в реальном времени:**
```bash
docker stats                        # все контейнеры
docker stats myapp                  # конкретный контейнер
docker stats --no-stream            # один снимок без обновления
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Информация выводится:**
- `CONTAINER` — имя/ID контейнера
- `CPU %` — использование процессора
- `MEM USAGE / LIMIT` — использование памяти
- `NET I/O` — сетевой трафик (входящий/исходящий)
- `BLOCK I/O` — дисковый I/O (читание/запись)
- `PIDS` — количество процессов

---

#### Подробная информация о контейнере

**JSON формат (полная информация):**
```bash
docker inspect myapp                # весь JSON
docker inspect myapp | less         # постранично
```

**Извлечение конкретного поля:**
```bash
docker inspect --format '{{.State.Status}}' myapp        # статус
docker inspect --format '{{.State.Pid}}' myapp           # PID процесса
docker inspect --format '{{.HostConfig.Memory}}' myapp   # лимит памяти
docker inspect --format '{{.Config.Env}}' myapp          # переменные окружения
docker inspect --format '{{.NetworkSettings.IPAddress}}' myapp   # IP в сети
```

**Полезные поля:**
- `.State.Status` — текущий статус (running, exited, paused)
- `.State.Pid` — PID главного процесса контейнера
- `.State.StartedAt` — время запуска
- `.State.FinishedAt` — время остановки
- `.HostConfig.Memory` — лимит памяти (bytes)
- `.Config.Env` — переменные окружения
- `.NetworkSettings.IPAddress` — IP в сети Docker
- `.Mounts` — подключенные volumes

---

#### Работа с логами

**Просмотр всех логов:**
```bash
docker logs myapp                   # все логи
docker logs myapp 2>&1              # stdout + stderr
```

**Follow mode (в реальном времени, как tail -f):**
```bash
docker logs -f myapp                # следить за новыми логами
docker logs -f --tail=10 myapp      # последние 10 строк и дальше
```

**Параметры для фильтрации:**
```bash
docker logs --tail=100 myapp        # последние 100 строк
docker logs --since 10m myapp       # логи за последние 10 минут
docker logs --until 5m myapp        # логи до 5 минут назад
docker logs --timestamps myapp      # с временными метками (UTC)
```

---

#### Фильтрация логов с grep

**Поиск по ключевым словам:**
```bash
docker logs myapp | grep ERROR                          # только ошибки
docker logs myapp | grep -i warning                     # warnings (case-insensitive)
docker logs myapp | grep -c "error"                     # количество ошибок
```

**Контекст вокруг совпадения:**
```bash
docker logs myapp | grep -A 3 "error"                   # 3 строки ПОСЛЕ
docker logs myapp | grep -B 2 "error"                   # 2 строки ПЕРЕД
docker logs myapp | grep -A 2 -B 2 "error"              # по 2 строки со всех сторон
```

**Комбинирование команд:**
```bash
docker logs myapp | head -50                            # первые 50 строк
docker logs myapp | tail -20                            # последние 20 строк
docker logs myapp | wc -l                               # количество строк
docker logs myapp | sort | uniq -c | sort -rn           # топ повторяющихся строк
```

---

#### Сохранение логов

**В файл:**
```bash
docker logs myapp > app.log                             # все логи
docker logs myapp >> app.log                            # добавить в конец
docker logs myapp 2>&1 > app.log                        # stdout + stderr
```

**С фильтрацией:**
```bash
docker logs myapp | grep ERROR > errors.log             # только ошибки
docker logs myapp | tail -1000 > recent.log             # последние 1000 строк
docker logs -f myapp | tee app.log                      # выводить И сохранять одновременно
```

---

#### Выполнение команд в контейнере

**docker exec** — запустить команду в работающем контейнере:
```bash
docker exec <container> <command>
```

**Важно:** Контейнер должен быть в состоянии `Running`. На Paused или Stopped не работает.

---

#### Параметры docker exec

| Параметр | Описание |
|----------|---------|
| `-i, --interactive` | Keep STDIN open (интерактивный ввод) |
| `-t, --tty` | Allocate a pseudo-terminal (TTY для shell) |
| `-it` | Комбинированно для интерактивной shell |
| `-d, --detach` | Запустить в фоне (background) |
| `-e, --env` | Передать переменные окружения |
| `-u, --user` | Запустить от имени пользователя |
| `-w, --workdir` | Установить рабочую директорию |

---

#### Примеры docker exec

**Простые команды:**
```bash
docker exec myapp pwd                                   # текущая директория
docker exec myapp mongo --version                       # версия приложения
docker exec myapp ls /                                  # список файлов
docker exec myapp echo $HOME                            # переменная окружения
```

**Интерактивная shell:**
```bash
docker exec -it myapp bash                              # интерактивный bash
docker exec -it myapp sh                                # интерактивный sh (если bash нет)
```

**С переменными окружения:**
```bash
docker exec -e VAR=value myapp printenv VAR             # одна переменная
docker exec -e VAR1=val1 -e VAR2=val2 myapp env         # несколько переменных
docker exec myapp env                                   # все переменные контейнера
```

**От другого пользователя:**
```bash
docker exec -u root myapp whoami                        # от root
docker exec -u postgres myapp psql --version            # от postgres
```

**В определённой директории:**
```bash
docker exec -w /app myapp ls                            # выполнить в /app
```

**Фоновое выполнение:**
```bash
docker exec -d myapp python script.py                   # запустить в фоне
docker exec -d myapp bash -c 'command > /tmp/out.log'   # с перенаправлением
```

---

#### Отладка контейнера

**Процессы и система:**
```bash
docker exec myapp ps aux                                # список процессов
docker exec myapp top -b -n 1                           # один снимок top
docker exec myapp df -h                                 # использование диска
docker exec myapp du -sh /*                             # размер директорий
```

**Сеть:**
```bash
docker exec myapp netstat -tlnp                         # открытые порты
docker exec myapp ss -tlnp                              # современнее чем netstat
docker exec myapp curl localhost:8080                   # проверить доступность
```

**Память:**
```bash
docker exec myapp free -h                               # информация о памяти
docker exec myapp cat /proc/meminfo                     # подробно
```

---

#### Практическое упражнение: MongoDB

```bash
# 1. Создание контейнера
docker create --name mongodb mongo:latest

# 2. Запуск
docker start mongodb

# 3. Создание файла в контейнере
docker exec mongodb touch /myfile.txt

# 4. Проверка сохранности (остановка и запуск)
docker stop mongodb
docker start mongodb
docker exec mongodb ls /myfile.txt   # ✅ файл есть!

# 5. Интерактивный доступ
docker exec -it mongodb bash
# Внутри контейнера: можешь выполнять команды

# 6. Просмотр логов
docker logs mongodb | head -20

# 7. Статистика
docker stats mongodb --no-stream

# 8. Удаление
docker rm -f mongodb
```

---

#### Полезные команды отладки

```bash
# Комбинированная отладка
docker stats myapp --no-stream                          # снимок ресурсов
docker logs myapp | tail -50 | grep -i error            # последние 50 строк с ошибками
docker logs -f --tail=20 myapp                          # 20 последних строк в реальном времени
docker inspect myapp | grep -A 5 "State"                # информация о состоянии
docker exec myapp bash -c 'ps aux && netstat -tlnp'     # процессы и порты
```

---