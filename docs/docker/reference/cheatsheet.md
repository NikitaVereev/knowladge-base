---
title: "Docker CLI Cheatsheet"
type: reference
tags: [docker, cli, commands, cheatsheet]
sources:
  docs: "https://docs.docker.com/reference/cli/docker/"
related:
  - "[[docker/reference/dockerfile]]"
  - "[[docker/reference/compose-spec]]"
  - "[[docker/how-to/debug-containers]]"
---

# Шпаргалка по Docker CLI

> **Справочник:** Все команды Docker CLI на одной странице. Ctrl+F для поиска.

## Containers — жизненный цикл

| Команда | Описание |
| :--- | :--- |
| `docker run [opts] <image>` | Создать и запустить контейнер |
| `docker run -d --name app nginx` | Запустить в фоне с именем |
| `docker run --rm -it alpine sh` | Одноразовый интерактивный контейнер (удалится после выхода) |
| `docker start <container>` | Запустить остановленный контейнер |
| `docker stop <container>` | Остановить: SIGTERM → 10s → SIGKILL |
| `docker stop -t 30 <container>` | Остановить с таймаутом 30 сек |
| `docker kill <container>` | Убить немедленно (SIGKILL) |
| `docker restart <container>` | Перезапустить |
| `docker pause / unpause` | Заморозить / разморозить процессы (SIGSTOP/SIGCONT) |
| `docker rm <container>` | Удалить остановленный контейнер |
| `docker rm -f <container>` | Удалить работающий контейнер |
| `docker rm $(docker ps -aq)` | Удалить все остановленные контейнеры |
| `docker rename old new` | Переименовать контейнер |
| `docker wait <container>` | Ждать завершения, вернуть exit code |
| `docker update --memory=1g <c>` | Обновить лимиты на лету |

### Ключевые флаги `docker run`

```bash
docker run -d \                     # фон (detached)
  --name app \                      # имя контейнера
  --rm \                            # удалить после остановки
  -p 127.0.0.1:8080:80 \           # порт: только localhost
  -p 3000:3000 \                    # порт: все интерфейсы
  -v mydata:/app/data \             # named volume
  -v $(pwd)/src:/app/src \          # bind mount (код → hot reload)
  --tmpfs /tmp:size=100m \          # tmpfs в RAM
  -e NODE_ENV=production \          # переменная окружения
  --env-file .env \                 # переменные из файла
  --network mynet \                 # подключить к сети
  --restart unless-stopped \        # политика перезапуска
  --user 1001:1001 \               # запуск от UID:GID
  --read-only \                     # read-only корневая ФС
  --cap-drop=ALL \                  # сбросить все capabilities
  --security-opt no-new-privileges \# запрет повышения привилегий
  --memory=512m \                   # лимит RAM
  --cpus=1.0 \                      # лимит CPU
  --pids-limit=100 \                # лимит процессов
  --init \                          # tini как PID 1 (правильные сигналы)
  --health-cmd "wget -q --spider http://localhost:3000" \
  --health-interval 30s \
  --health-timeout 5s \
  myapp:latest
```

## Containers — операции и отладка

| Команда | Описание |
| :--- | :--- |
| `docker ps` | Запущенные контейнеры |
| `docker ps -a` | Все контейнеры (включая остановленные) |
| `docker ps -q` | Только ID (для скриптов) |
| `docker ps --filter status=exited` | Только завершённые |
| `docker ps --format '{{.Names}}\t{{.Status}}'` | Кастомный формат |
| `docker logs <container>` | Логи (STDOUT + STDERR) |
| `docker logs -f <container>` | Логи в реальном времени |
| `docker logs --since 1h` | Логи за последний час |
| `docker logs --tail 100` | Последние 100 строк |
| `docker exec -it <c> sh` | Shell внутри контейнера |
| `docker exec -it <c> /bin/bash` | Bash внутри (если есть) |
| `docker exec -u root <c> sh` | Shell от root (даже если USER != root) |
| `docker exec <c> env` | Показать переменные окружения |
| `docker cp <c>:/app/log.txt .` | Скопировать файл из контейнера |
| `docker cp ./fix.sh <c>:/tmp/` | Скопировать файл в контейнер |
| `docker inspect <container>` | Полный JSON конфигурации |
| `docker stats` | CPU/RAM/Net/IO всех контейнеров (live) |
| `docker stats --no-stream` | Один снимок (для скриптов) |
| `docker top <container>` | Процессы внутри контейнера |
| `docker diff <container>` | Изменённые файлы (A=add, C=change, D=delete) |
| `docker port <container>` | Показать маппинг портов |
| `docker attach <container>` | Подключиться к STDOUT (Ctrl+P,Q для отсоединения) |

### Полезные `--format` шаблоны

```bash
# IP-адрес контейнера
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <c>

# Exit code
docker inspect -f '{{.State.ExitCode}}' <c>

# OOM Killed?
docker inspect -f '{{.State.OOMKilled}}' <c>

# Healthcheck статус
docker inspect -f '{{.State.Health.Status}}' <c>

# Все переменные окружения
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' <c>

# Bind mounts
docker inspect -f '{{range .Mounts}}{{.Source}} → {{.Destination}}{{println}}{{end}}' <c>

# Время запуска
docker inspect -f '{{.State.StartedAt}}' <c>

# Restart count
docker inspect -f '{{.RestartCount}}' <c>

# Таблица: имя, статус, порты
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## Images

| Команда | Описание |
| :--- | :--- |
| `docker images` | Список локальных образов |
| `docker images -q` | Только ID |
| `docker images --filter dangling=true` | Только `<none>` образы |
| `docker pull <image>` | Скачать из Registry |
| `docker pull nginx:1.25-alpine` | Конкретная версия |
| `docker push <image>` | Отправить в Registry |
| `docker build -t app:v1 .` | Собрать из Dockerfile |
| `docker build -f Dockerfile.prod .` | Указать файл |
| `docker build --target runner .` | Собрать до конкретного stage |
| `docker build --no-cache .` | Без кэша (полная пересборка) |
| `docker tag src:v1 registry/src:v1` | Добавить тег |
| `docker rmi <image>` | Удалить образ |
| `docker rmi $(docker images -q -f dangling=true)` | Удалить все `<none>` |
| `docker history <image>` | Слои образа с размерами |
| `docker history --no-trunc <image>` | Полные команды слоёв |
| `docker save -o app.tar app:v1` | Экспорт в tar (оффлайн перенос) |
| `docker load -i app.tar` | Импорт из tar |
| `docker image prune` | Удалить dangling образы |
| `docker image prune -a` | Удалить все неиспользуемые |

### BuildKit и Buildx

```bash
# Включить BuildKit (если не по умолчанию)
export DOCKER_BUILDKIT=1

# Сборка с инлайн-кэшем
docker build --build-arg BUILDKIT_INLINE_CACHE=1 .

# Сборка для другой платформы
docker buildx build --platform linux/amd64,linux/arm64 -t app:v1 --push .

# Просмотр билдеров
docker buildx ls

# Создать новый билдер (для multi-platform)
docker buildx create --name mybuilder --use
docker buildx inspect --bootstrap

# Кэш в GitHub Actions
docker buildx build \
  --cache-from type=gha \
  --cache-to type=gha,mode=max \
  -t app:latest --push .

# Очистка кэша сборщика
docker builder prune
docker builder prune -a    # весь кэш
```

## Volumes

| Команда | Описание |
| :--- | :--- |
| `docker volume create mydata` | Создать том |
| `docker volume ls` | Список томов |
| `docker volume inspect mydata` | Детали (путь на хосте) |
| `docker volume rm mydata` | Удалить (данные потеряются!) |
| `docker volume prune` | Удалить все неиспользуемые |

### Практические примеры

```bash
# Просмотреть содержимое volume
docker run --rm -v mydata:/data alpine ls -la /data

# Бэкап volume в tar
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar czf /backup/mydata-backup.tar.gz -C /data .

# Восстановление volume из tar
docker run --rm -v mydata:/data -v $(pwd):/backup alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /data

# Копирование volume → volume
docker run --rm -v src:/from -v dst:/to alpine sh -c 'cp -a /from/. /to/'

# Узнать размер volume
docker system df -v | grep mydata
```

## Networks

| Команда | Описание |
| :--- | :--- |
| `docker network create mynet` | Создать bridge-сеть |
| `docker network create -d overlay mynet` | Overlay (Swarm) |
| `docker network create --subnet 172.28.0.0/16 mynet` | С кастомной подсетью |
| `docker network ls` | Список сетей |
| `docker network inspect mynet` | Детали: подсеть, контейнеры |
| `docker network connect mynet <c>` | Подключить контейнер к сети |
| `docker network disconnect mynet <c>` | Отключить от сети |
| `docker network rm mynet` | Удалить сеть |
| `docker network prune` | Удалить неиспользуемые сети |

### Диагностика сети

```bash
# К каким сетям подключён контейнер
docker inspect <c> -f '{{json .NetworkSettings.Networks}}' | python3 -m json.tool

# Какие контейнеры в сети
docker network inspect mynet \
  -f '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{println}}{{end}}'

# DNS-резолвинг изнутри
docker exec <c> nslookup db

# Пинг между контейнерами
docker exec app ping -c 2 db

# Тяжёлая артиллерия — netshoot
docker run --rm -it --network mynet nicolaka/netshoot
# Внутри: dig, nslookup, tcpdump, iperf, curl, nmap

# Проверить iptables-правила Docker
sudo iptables -t nat -L DOCKER -n
sudo iptables -L DOCKER-USER -n
```

## System & Cleanup

| Команда | Описание |
| :--- | :--- |
| `docker info` | Системная информация |
| `docker version` | Версия клиента и сервера |
| `docker system df` | Использование диска |
| `docker system df -v` | Детально (каждый образ/volume) |
| `docker system prune` | Удалить: stopped containers + dangling images + unused networks |
| `docker system prune -a` | + все неиспользуемые образы |
| `docker system prune -a --volumes` | + неиспользуемые volumes (⚠ осторожно!) |
| `docker system events` | Поток событий в реальном времени |
| `docker system events --filter type=container` | Только события контейнеров |
| `docker builder prune` | Очистить кэш BuildKit |
| `docker builder prune -a` | Весь кэш (может освободить десятки GB) |

### Сколько занимает Docker

```bash
# Общая сводка
docker system df

# Топ-10 образов по размеру
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -hr | head -10

# Размер логов каждого контейнера
for c in $(docker ps -q); do
  name=$(docker inspect -f '{{.Name}}' $c | sed 's|^/||')
  log=$(docker inspect -f '{{.LogPath}}' $c)
  size=$(du -sh "$log" 2>/dev/null | awk '{print $1}')
  echo "$size $name"
done | sort -hr
```

## Docker Compose

| Команда | Описание |
| :--- | :--- |
| `docker compose up -d` | Собрать + запустить в фоне |
| `docker compose up --build` | Пересобрать образы перед запуском |
| `docker compose up --force-recreate` | Пересоздать контейнеры |
| `docker compose down` | Остановить + удалить контейнеры и сети |
| `docker compose down -v` | + удалить volumes (⚠ данные!) |
| `docker compose down --rmi all` | + удалить образы |
| `docker compose logs -f` | Логи всех сервисов (live) |
| `docker compose logs -f app` | Логи одного сервиса |
| `docker compose ps` | Статус сервисов |
| `docker compose exec app sh` | Shell внутри сервиса |
| `docker compose run --rm app npm test` | Одноразовый запуск команды |
| `docker compose build` | Только собрать, не запускать |
| `docker compose pull` | Скачать образы из registry |
| `docker compose config` | Валидация + итоговый YAML |
| `docker compose config --services` | Список сервисов |
| `docker compose top` | Процессы всех сервисов |
| `docker compose cp app:/log.txt .` | Копировать файл из сервиса |
| `docker compose --profile debug up` | Запустить с профилем |
| `docker compose -f compose.yaml -f compose.prod.yaml up` | Merge нескольких файлов |
| `docker compose up -d --scale app=3` | 3 реплики сервиса |
| `docker compose up -d --no-deps --build app` | Обновить один сервис |
| `docker compose restart app` | Перезапустить без пересоздания |

## Однострочники (One-Liners)

```bash
# Остановить все контейнеры
docker stop $(docker ps -q)

# Удалить все остановленные контейнеры
docker container prune -f

# Удалить все <none> образы
docker image prune -f

# Удалить ВСЁ (контейнеры, образы, volumes, networks, cache)
docker system prune -a --volumes -f

# IP всех запущенных контейнеров
docker ps -q | xargs -I{} docker inspect \
  -f '{{.Name}} {{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' {}

# Контейнеры с exit code != 0
docker ps -a --filter "exited=1" --format "table {{.Names}}\t{{.Status}}"

# Контейнеры, использующие больше всего RAM
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}" | sort -k2 -hr

# Следить за событиями
docker events --filter type=container \
  --format '{{.Time}} {{.Action}} {{.Actor.Attributes.name}}'

# Найти контейнер по порту
docker ps --filter "publish=8080"

# Перезапустить все unhealthy контейнеры
docker ps --filter health=unhealthy -q | xargs -r docker restart

# Обнулить логи контейнера
truncate -s 0 $(docker inspect -f '{{.LogPath}}' <container>)

# Сколько слоёв в образе
docker inspect <image> --format '{{len .RootFS.Layers}}'

# Дата сборки образа
docker inspect <image> --format '{{.Created}}'

# Экспорт переменных из контейнера
docker exec app env | grep -E '^(DB_|REDIS_|API_)' > .env.extracted
```

## daemon.json — рекомендуемые настройки

Файл: `/etc/docker/daemon.json` (создать если нет). После изменения: `sudo systemctl restart docker`.

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {"base": "172.17.0.0/12", "size": 24}
  ],
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false
}
```

| Параметр | Зачем |
|----------|-------|
| `log-opts` | Ротация логов — без этого Docker заполнит диск |
| `default-address-pools` | Избежать конфликтов подсетей с VPN/LAN |
| `no-new-privileges` | Запрет SUID/SGID для всех контейнеров |
| `live-restore` | Контейнеры продолжают работать при рестарте демона |
| `userland-proxy: false` | Отключить docker-proxy, только iptables — быстрее |
