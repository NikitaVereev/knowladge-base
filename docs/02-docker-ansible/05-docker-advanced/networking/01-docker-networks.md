### Docker Network: Основы и режимы

Сетевое взаимодействие контейнеров Docker.

---

#### Основы Docker Network

**Docker Network:**
- Управляется открытой библиотекой **LibNetwork** (написана на Go)
- Обеспечивает функциональности для изоляции и коммуникации контейнеров
- Интегрирована в Docker Engine

**Компоненты под капотом:**
- **Network Namespaces** — Linux изоляция сетей
- **Linux Bridge** — виртуальный коммутатор для контейнеров
- **Virtual Ethernet Devices** — виртуальные сетевые интерфейсы (veth pairs)
- **IPTables** — правила маршрутизации и NAT

**Диаграмма:**
```
┌────────────────────────────────────────────┐
│          Docker Host (Linux)               │
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────┐    ┌──────────┐             │
│  │Container │    │Container │             │
│  │    1     │    │    2     │             │
│  │ eth0     │    │ eth0     │             │
│  └────┬─────┘    └────┬─────┘             │
│       │ veth0         │ veth1             │
│       │               │                   │
│       └───────┬───────┘                   │
│               │                           │
│          ┌─────────┐                      │
│          │  Bridge │                      │
│          │ docker0 │                      │
│          └────┬────┘                      │
│               │                           │
│          ┌────────────────┐               │
│          │ eth0 (хоста)   │               │
│          └────────────────┘               │
└────────────────────────────────────────────┘
```

---

#### Типы сетевых драйверов

| Драйвер | Использование | Примечание |
|---------|---------------|-----------|
| **Bridge** | Контейнеры на одном хосте | По умолчанию, лучше для development |
| **Host** | Прямой доступ к сети хоста | Нет изоляции, высокая производительность |
| **Overlay** | Контейнеры на разных хостах | Для Docker Swarm, Kubernetes |
| **Macvlan** | Уникальный MAC адрес | Редко используется |
| **None** | Без сети | Для контейнеров которым не нужна сеть |

---

#### Bridge Network (по умолчанию)

**Дефолтная сеть:**
```bash
docker network ls
# NAME              DRIVER    SCOPE
# bridge            bridge    local
# host              host      local
# none              null      local
```

**Как работает:**
1. При запуске контейнера `docker run` без флага `--network` используется дефолтный bridge
2. Контейнер получает виртуальный Ethernet интерфейс
3. Получает IP адрес из подсети сети (обычно 172.17.0.0/16)
4. Может обмениваться данными с другими контейнерами в той же сети

**Пример:**
```bash
# Запустить контейнер (использует дефолтный bridge)
docker run -d --name web nginx:latest

# Запустить второй контейнер
docker run -d --name db postgres:latest

# web может достучаться до db
docker exec web ping db        # ✅ работает (Service Discovery)
```

---

#### Service Discovery

**Встроенный DNS:**
- Docker имеет встроенный DNS сервер (по адресу 127.0.0.11:53)
- Контейнеры могут обращаться друг к другу **по имени**
- Имя контейнера автоматически разрешается в IP адрес

**Как работает:**
```bash
# Container 1 (web)
docker run -d --name web nginx:latest
# IP: 172.17.0.2

# Container 2 (app)
docker run -d --name app node:latest
# IP: 172.17.0.3

# Из app можем достучаться до web:
docker exec app curl http://web:80    # ✅ работает!
# Docker разрешил "web" → 172.17.0.2

# Но это НЕ работает в дефолтной bridge!
# Только в пользовательских bridge сетях
```

**Важно:** Service Discovery работает только в **пользовательских bridge сетях**, не в дефолтной!

---

#### Порт-маппинг (Port Publishing)

**Зачем нужно:**
- Контейнеры изолированы в своей сети
- Для доступа извне нужно пробросить порт

**Синтаксис:**
```bash
docker run -p <хост_порт>:<контейнер_порт> <образ>

# Примеры:
docker run -p 8080:80 nginx              # хост:8080 → контейнер:80
docker run -p 5432:5432 postgres         # хост:5432 → контейнер:5432
docker run -p 3000:3000 node:latest      # хост:3000 → контейнер:3000
docker run -p 127.0.0.1:5432:5432 postgres  # только localhost
```

**Как работает:**
```
Внешний трафик → localhost:8080 → IPTables NAT → контейнер:80
```

**Проверка маппинга:**
```bash
docker ps
# PORTS: 0.0.0.0:8080->80/tcp

# Или через inspect:
docker inspect <container_id> | grep PortBindings
```

---

#### Создание пользовательской Bridge сети

**Зачем:**
- Service Discovery по имени
- Изоляция контейнеров от остальных
- Лучший контроль над сетью

**Создание:**
```bash
docker network create mynet
docker network create --subnet=172.20.0.0/16 mynet
docker network create -d bridge --subnet=192.168.0.0/16 mynet
```

**Параметры:**
- `-d, --driver` — тип драйвера (по умолчанию bridge)
- `--subnet` — подсеть IP (CIDR)
- `--ip-range` — диапазон IP для allocation
- `--gateway` — шлюз по умолчанию
- `--label` — метаданные

**Запуск контейнеров в сети:**
```bash
# Способ 1: При создании
docker run -d --name web --network mynet nginx:latest

# Способ 2: Подключить существующий контейнер
docker network connect mynet web
```

**Проверка:**
```bash
docker network inspect mynet
```

**Результат:**
```json
{
  "Containers": {
    "abc123...": {
      "Name": "web",
      "IPv4Address": "172.20.0.2/16"
    },
    "def456...": {
      "Name": "db",
      "IPv4Address": "172.20.0.3/16"
    }
  }
}
```

---

#### Host Network

**Что делает:**
- Убирает слой сетевой абстракции
- Контейнер напрямую использует сеть хоста
- Нет виртуальных Ethernet интерфейсов
- Нет необходимости в проброске портов

**Когда использовать:**
- Мониторинг системы (нужен доступ ко всем портам)
- Высокопроизводительные приложения (меньше overhead)
- Прямой доступ к сервисам на хосте

**Пример:**
```bash
# Контейнер с host сетью
docker run -d --name nginx --network host nginx:latest

# Nginx слушает на хосте порт 80 напрямую (без маппинга)
curl localhost:80    # ✅ работает

# Но контейнер может обращаться к localhost:3000 если там что-то работает
docker exec nginx curl localhost:3000   # ✅ доступ к хосту
```

**Ограничения:**
- Невозможно использовать с `-p` флагом (не нужно)
- Потенциальная уязвимость (контейнер видит всю сеть хоста)

---

#### None Network

**Что делает:**
- Контейнер создаётся БЕЗ сетевого доступа
- Нет Ethernet интерфейсов (кроме loopback)
- Экономит ресурсы

**Когда использовать:**
- Batch jobs (обработка данных без сети)
- Контейнеры которые работают offline
- Контейнеры для фоновых вычислений

**Пример:**
```bash
# Контейнер без сети
docker run -d --name worker --network none python:latest python batch_process.py

# Внутри контейнера:
docker exec worker ifconfig
# Только lo (loopback), нет eth0
```

---

#### Overlay Network (для Swarm/Kubernetes)

**Для чего:**
- Соединение контейнеров на РАЗНЫХ хостах
- Используется в Docker Swarm и Kubernetes
- Создаёт виртуальную сеть поверх физической

**Пример (Docker Swarm):**
```bash
docker swarm init
docker network create --driver overlay myoverlay

# Контейнеры на разных нодах могут обмениваться данными
docker service create --network myoverlay web-service nginx
```

**Но это более продвинутая тема для раздела про Orchestration.**

---

#### Практические примеры

##### Пример 1: Контейнеры в одной сети

```bash
# Создание сети
docker network create myapp

# Запуск web контейнера
docker run -d --name web --network myapp -p 8080:80 nginx

# Запуск app контейнера
docker run -d --name app --network myapp node:latest

# Из app можно обращаться к web:
docker exec app curl http://web:80     # ✅ работает

# С хоста можно обращаться через маппированный порт:
curl localhost:8080                    # ✅ работает
```

##### Пример 2: Database и Application

```bash
# Создание сети
docker network create dbnet

# Запуск PostgreSQL
docker run -d \
  --name postgres \
  --network dbnet \
  -e POSTGRES_PASSWORD=secret \
  postgres:latest

# Запуск приложения
docker run -d \
  --name app \
  --network dbnet \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://postgres:secret@postgres:5432/mydb \
  myapp:latest

# Приложение может подключиться к БД:
docker exec app psql -h postgres -U postgres -d mydb   # ✅ работает
```

##### Пример 3: Host Network для мониторинга

```bash
# Prometheus с доступом к хосту
docker run -d \
  --name prometheus \
  --network host \
  prom/prometheus:latest

# Prometheus может скрейпить localhost:9090 напрямую
```

---

#### Управление сетями

**Команды:**
```bash
docker network ls                          # список всех сетей
docker network create mynet                # создание сети
docker network inspect mynet               # информация о сети
docker network connect mynet container     # подключить контейнер
docker network disconnect mynet container  # отключить контейнер
docker network rm mynet                    # удалить сеть
docker network prune                       # удалить неиспользуемые
```

---

#### DNS в Docker

**Встроенный DNS:**
- IP: `127.0.0.11:53`
- Автоматически разрешает имена контейнеров
- Работает в пользовательских bridge сетях

**Кастомный DNS:**
```bash
docker run --dns 8.8.8.8 --dns 8.8.4.4 <image>

# Проверка:
docker exec <container> cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 8.8.4.4
```

**Использование:**
- Corporate DNS для резолвинга внутренних доменов
- Google DNS (8.8.8.8) для публичных
- Cloudflare DNS (1.1.1.1) как альтернатива

---

#### Проблемы и решения

**Проблема: Контейнеры не видят друг друга по имени**
```bash
# Решение: используй пользовательскую bridge сеть
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet curl http://web   # ✅ работает
```

**Проблема: Port уже занят**
```bash
# Решение: используй другой хост-порт
docker run -p 8081:80 nginx   # вместо 8080

# Или найди кто занимает порт:
lsof -i :8080
```

**Проблема: Контейнер не может достучаться в интернет**
```bash
# Решение: проверь что контейнер в сети (не none)
docker run --network bridge <image>   # default bridge

# Или кастомная:
docker network create mynet
docker run --network mynet <image>
```
