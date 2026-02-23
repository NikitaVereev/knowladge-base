---
title: "Сокеты: сетевые и Unix domain"
type: explanation
tags: [linux, sockets, network, ipc, unix-domain, tcp, udp, docker, systemd]
sources:
  docs: "https://man7.org/linux/man-pages/man7/socket.7.html"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 10.9–10.10"
related:
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/process-model]]"
  - "[[linux/explanation/systemd]]"
  - "[[linux/how-to/network-diagnostics]]"
  - "[[docker/explanation/architecture]]"
---

# Сокеты: сетевые и Unix domain

> **TL;DR:** Сокет — интерфейс, через который процесс получает доступ к сети (или общается с другим процессом). Сетевые сокеты (`SOCK_STREAM`, `SOCK_DGRAM`) работают поверх TCP/UDP. Unix domain sockets — то же API, но без сети: через файл в файловой системе, быстрее и с контролем доступа через права. Почему это важно: `docker.sock`, D-Bus, MySQL-сокет, systemd socket activation — всё это сокеты.

## Зачем это знать

Сокеты — **граница между пространством пользователя и ядром** для всего, что связано с сетью и межпроцессным взаимодействием (IPC). Без понимания сокетов непонятно:

- Почему Docker требует доступ к `/var/run/docker.sock` и чем это опасно
- Как systemd реализует socket activation (слушает порт → запускает сервис по требованию)
- Почему MySQL быстрее работает через `/var/run/mysqld/mysqld.sock`, чем через `localhost:3306`
- Что показывает `ss -tlnp` — это и есть список сокетов в состоянии LISTEN
- Как сетевой сервер (sshd, nginx) обрабатывает тысячи подключений одновременно

## Сетевые сокеты

Когда процесс хочет отправить или получить данные по сети, он создаёт сокет — специальный объект ядра. Через сокет процесс выполняет системные вызовы `send(2)` и `recv(2)` для обмена данными.

### Типы сокетов

| Тип | Константа | Транспорт | Свойства |
|-----|-----------|-----------|---------|
| Потоковый | `SOCK_STREAM` | TCP | Надёжный, упорядоченный поток байтов. Требует соединения |
| Датаграммный | `SOCK_DGRAM` | UDP | Отдельные сообщения, без гарантий доставки и порядка |
| Raw | `SOCK_RAW` | IP напрямую | Для диагностики (ping, tcpdump). Нужен root или `CAP_NET_RAW` |

Тип сокета определяет поведение: если нужно переключить приложение с TCP на UDP — меняется в основном код инициализации, а код отправки/получения данных остаётся похожим. Это и есть главная гибкость сокетного API.

### Серверный workflow: listen → accept → fork

Большинство сетевых серверов работают по одной схеме. Сервер использует **два вида сокетов**: один для прослушивания, другой для обмена данными с конкретным клиентом.

```
Главный процесс
      │
      ▼
  socket()          ← создать сокет
      │
      ▼
  bind(port)        ← привязать к порту (например, :22)
      │
      ▼
  listen()          ← начать прослушивание
      │
      ▼
  ┌─► accept()      ← ждать подключения (блокирующий вызов)
  │       │
  │       ▼
  │   fork()         ← создать дочерний процесс
  │    ┌──┴──┐
  │    │     │
  │  Parent  Child
  │    │     │
  │    │   recv()/send()  ← обмен данными с клиентом
  │    │     │
  │    │   close()        ← закрыть соединение, exit
  │    │
  └────┘                  ← вернуться к accept()
```

Именно так работает sshd: master-процесс слушает порт 22, при подключении форкается, дочерний процесс обслуживает сессию. Подробнее: [[ssh/explanation/ssh-service-architecture]].

> **Вариации модели:** fork-per-connection — простая, но `fork()` создаёт накладные расходы. Высоконагруженные серверы (nginx, Apache) используют **пул рабочих процессов**, созданных заранее, или асинхронные модели (`epoll`, `io_uring`). UDP-серверы вообще не форкаются — им не нужно поддерживать соединение, они просто получают пакеты и отвечают.

### Что идентифицирует соединение

TCP-соединение однозначно определяется четвёркой:

```
src_ip:src_port ↔ dst_ip:dst_port
```

Поэтому один серверный порт (например, :80) может обслуживать тысячи клиентов одновременно — каждое соединение уникально за счёт разных `src_ip:src_port`.

## Unix domain sockets (UDS)

Процессам на **одной машине** не обязательно использовать сеть для общения. Unix domain socket — тот же сокетный API (`listen`, `accept`, `send`, `recv`), но **без сетевого стека**. Данные передаются напрямую через ядро, минуя все сетевые уровни.

### Зачем нужны, если есть localhost

Два преимущества:

**Производительность.** Ядру не нужно проходить через TCP/IP-стек (заголовки, checksums, маршрутизация). На практике UDS значительно быстрее, чем TCP через `127.0.0.1`.

**Контроль доступа через файловую систему.** UDS представлен файлом (тип `s`), и к нему применяются стандартные Unix-права:

```bash
ls -l /var/run/dbus/system_bus_socket
# srwxrwxrwx 1 root root 0 Nov 9 08:52 /var/run/dbus/system_bus_socket
#  ↑
#  s = socket file
```

Процесс без прав на файл сокета не сможет к нему подключиться. Это проще и безопаснее, чем настройка сетевого firewall для localhost-соединений.

### Где UDS встречаются на практике

| Файл сокета | Сервис | Зачем |
|-------------|--------|-------|
| `/var/run/docker.sock` | Docker daemon | CLI и API общаются с dockerd |
| `/var/run/dbus/system_bus_socket` | D-Bus | Системная шина сообщений (systemd, NetworkManager) |
| `/var/run/mysqld/mysqld.sock` | MySQL/MariaDB | Локальные подключения (быстрее TCP) |
| `/var/run/postgresql/.s.PGSQL.5432` | PostgreSQL | Локальные подключения |
| `/run/containerd/containerd.sock` | containerd | Runtime контейнеров |
| `/run/snapd.socket` | snapd | Snap-пакеты |

> **Безымянные сокеты.** Не все UDS привязаны к файлам. Процесс может создать безымянный сокет и передать его дескриптор дочернему процессу. В выводе `lsof -U` такие сокеты отображаются как `socket` в столбце NAME.

### Диагностика UDS

```bash
# Все Unix domain sockets в системе
sudo lsof -U
# COMMAND   PID    USER  FD  TYPE  NODE NAME
# mysqld    19701  mysql 12u unix  35201227 /var/run/mysqld/mysqld.sock
# chromium  26534  juser 5u  unix  42445141 socket        ← безымянный
# tlsmgr    30480  postfix 5u unix 17009106 socket

# Только серверные (слушающие) UDS
ss -xlnp
# Netid  State  Recv-Q  Send-Q  Local Address:Port
# u_str  LISTEN 0       128     /var/run/docker.sock

# Конкретный файл сокета
sudo lsof /var/run/docker.sock
```

### UDS и Docker: важное предупреждение

Доступ к `/var/run/docker.sock` = **полный контроль над Docker daemon** = фактически root на хосте. Монтирование этого сокета в контейнер (`-v /var/run/docker.sock:/var/run/docker.sock`) даёт контейнеру возможность создавать другие контейнеры, монтировать любые директории хоста, запускать привилегированные процессы.

```bash
# Кто имеет доступ к Docker-сокету
ls -la /var/run/docker.sock
# srw-rw---- 1 root docker 0 ... /var/run/docker.sock
#                    ↑
#                    группа docker = root-equivalent
```

## Сетевые сокеты vs Unix domain: когда что

| Критерий | Сетевой сокет (TCP/UDP) | Unix domain socket |
|----------|------------------------|--------------------|
| Процессы на разных машинах | ✅ Единственный вариант | ❌ Только один хост |
| Производительность (один хост) | Средняя (TCP/IP overhead) | Высокая (нет сетевого стека) |
| Контроль доступа | Firewall, listen address | Права файловой системы |
| Требует настройки сети | Да | Нет |
| Типичное применение | Веб-серверы, SSH, API | D-Bus, Docker, БД (локально) |

Многие серверы предлагают **оба варианта**. MySQL принимает подключения и через TCP (:3306 для удалённых клиентов), и через UDS (`/var/run/mysqld/mysqld.sock` для локальных). PostgreSQL — аналогично.

## Сокеты и systemd: socket activation

systemd использует сокеты для запуска сервисов **по требованию**. Идея: systemd сам создаёт и слушает сокет, а при входящем подключении запускает сервис и передаёт ему готовый сокет.

```
Обычный режим:             Socket activation:
                           
sshd запущен всегда        systemd слушает :22
       │                          │
    listen(:22)              Подключение!
       │                          │
    accept()                systemd запускает sshd
       │                          │
    fork()                  sshd получает готовый сокет
```

Это работает и для сетевых сокетов (TCP/UDP), и для Unix domain sockets. В systemd это настраивается через `.socket`-юниты:

```ini
# /etc/systemd/system/myapp.socket
[Socket]
ListenStream=/run/myapp.sock    # UDS
# или
ListenStream=8080               # TCP порт

[Install]
WantedBy=sockets.target
```

Подробнее о юнитах: [[linux/explanation/systemd]].

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| `Permission denied` при подключении к UDS | Нет прав на файл сокета | Проверить `ls -la socket_path`, добавить пользователя в нужную группу |
| `Connection refused` на localhost | Порт слушается, но на другом интерфейсе | `ss -tlnp` — проверить bind address (`127.0.0.1` vs `0.0.0.0`) |
| `Address already in use` при старте сервера | Предыдущий процесс не освободил порт | `lsof -i :PORT` → kill; или `SO_REUSEADDR` в коде |
| Stale socket file | Файл UDS остался после краша сервиса | `rm /run/myapp.sock` и перезапуск |
| Docker-контейнер с `docker.sock` | Root-equivalent доступ | Не монтировать в продакшене, использовать Docker API через TCP с TLS |

## Связанные материалы

- [[linux/explanation/networking]] — стек TCP/IP, уровни, порты, путь пакета
- [[linux/explanation/process-model]] — fork, демоны, IPC-контекст
- [[linux/explanation/systemd]] — юниты, socket activation, targets
- [[linux/how-to/network-diagnostics]] — ss, lsof, tcpdump — диагностика сокетов на практике
- [[docker/explanation/architecture]] — Docker daemon и docker.sock
