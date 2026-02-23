---
title: "Диагностика сетевых служб"
type: how-to
tags: [linux, network, diagnostics, lsof, tcpdump, netcat, nmap, ports, debugging]
sources:
  docs: "https://man7.org/linux/man-pages/man8/lsof.8.html"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 10.5"
related:
  - "[[linux/how-to/monitor-system]]"
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/explanation/networking]]"
  - "[[ssh/explanation/ssh-service-architecture]]"
---

# Диагностика сетевых служб

> **TL;DR:** `ss -tlnp` — кто слушает. `lsof -iTCP -sTCP:LISTEN` — то же, но с деталями процессов.
> `tcpdump` — что летит по сети. `netcat` — ручное подключение к порту. `nmap` — сканирование извне.

Все эти инструменты работают на **прикладном уровне**, но проникают глубже — на транспортный и сетевой, потому что всё на прикладном уровне в итоге сопоставляется с чем-то на более нижних уровнях.

## Предварительные условия

```bash
# Большинство утилит предустановлены. Если нет:
sudo apt install lsof tcpdump nmap netcat-openbsd   # Debian/Ubuntu
sudo pacman -S lsof tcpdump nmap openbsd-netcat     # Arch
```

## ss и netstat — быстрый обзор портов

`ss` (socket statistics) — современная замена `netstat`. Оба показывают состояние соединений на транспортном уровне.

```bash
# Все слушающие TCP-порты с процессами
ss -tlnp
# State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
# LISTEN  0       128     0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=1234,fd=3))
# LISTEN  0       511     0.0.0.0:80           0.0.0.0:*          users:(("nginx",pid=5678,fd=6))

# Все активные TCP-соединения
ss -tnp

# UDP
ss -ulnp

# Всё вместе: TCP + UDP, listening + established
ss -tunap
```

Ключевые флаги `ss`:

| Флаг | Значение |
|------|---------|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | Только listening (серверные сокеты) |
| `-n` | Числовые адреса (не резолвить DNS — быстрее) |
| `-p` | Показать процесс (нужен root для чужих процессов) |
| `-a` | Все состояния (listening + established + ...) |
| `-4` / `-6` | Только IPv4 / IPv6 |

> Если `ss` недоступен (старые системы, контейнеры), используйте `netstat` с теми же флагами: `netstat -tlnp`.

## lsof — кто использует порт

`lsof -i` показывает **процессы**, работающие с сетью. Преимущество перед `ss`: более детальная информация о процессе (PID, user, FD) и мощные фильтры.

### Все сетевые соединения

```bash
# От root — видны все процессы
sudo lsof -i
# COMMAND   PID   USER  FD  TYPE  DEVICE  SIZE/OFF NODE NAME
# rpcbind   700   root  6u  IPv4  10492   0t0      UDP  *:sunrpc         ← UDP-сервер (RPC)
# rpcbind   700   root  8u  IPv4  10508   0t0      TCP  *:sunrpc (LISTEN)
# avahi-dae 872   avahi 13u IPv4  21736   0t0      UDP  *:mdns           ← multicast DNS
# cupsd     1010  root  9u  IPv6  42321   0t0      TCP  ip6-localhost:ipp (LISTEN)  ← печать, IPv6
# ssh       14366 juser 3u  IPv4  38995   0t0      TCP  thishost:55457->server:ssh (ESTABLISHED)
# chromium  26534 juser 8r  IPv4  42525   0t0      TCP  thishost:41551->cdn:https (ESTABLISHED)

# Обычный пользователь видит только свои процессы.
# Root видит всё — от системных демонов до пользовательских соединений.

# -n — не резолвить DNS (быстрее!)
# -P — не резолвить имена портов из /etc/services (показывать числа)
sudo lsof -nP -i
```

### Фильтрация по порту

```bash
# Кто занимает порт 8080?
sudo lsof -i :8080

# Только TCP на порту 443
sudo lsof -i TCP:443

# Только UDP на порту 53 (DNS)
sudo lsof -i UDP:53
```

Полный синтаксис фильтра: `-i [протокол][@хост][:порт]`

Хост и порт могут быть именами или номерами. Можно использовать имена служб из `/etc/services` вместо номеров портов (например, `-iTCP:ssh` вместо `-iTCP:22`).

```bash
# Соединения к конкретному хосту
sudo lsof -i @192.168.1.100

# TCP-соединения к хосту на порт 22
sudo lsof -i TCP@192.168.1.100:22

# Только IPv6
sudo lsof -i6

# IPv6 + TCP + порт 443
sudo lsof -i6TCP:443
```

### Фильтрация по состоянию соединения

Самый полезный фильтр — показать только серверные процессы:

```bash
# Все процессы, СЛУШАЮЩИЕ TCP-порты
sudo lsof -iTCP -sTCP:LISTEN
# COMMAND  PID   USER  FD  TYPE  NODE NAME
# sshd     1234  root  3u  TCP   *:ssh (LISTEN)
# nginx    5678  root  6u  TCP   *:http (LISTEN)
# node     9012  app   21u TCP   *:3000 (LISTEN)

# Все УСТАНОВЛЕННЫЕ TCP-соединения
sudo lsof -iTCP -sTCP:ESTABLISHED

# UDP-серверы (UDP не имеет состояний — просто -iUDP)
sudo lsof -iUDP
```

> **Почему UDP отдельно?** UDP-серверы не «прослушивают» и не имеют состояния соединения, поэтому фильтр `-sTCP:LISTEN` к ним неприменим. Используйте `-iUDP` — обычно UDP-серверов в системе немного, и вычленить нужный несложно.

> **Почему `lsof`, а не `ss`?** Для быстрого ответа «кто на порту» — `ss -tlnp` быстрее. Но `lsof` незаменим, когда нужно фильтровать по состоянию, протоколу, хосту и порту одновременно, а также когда важен контекст процесса (user, fd).

## tcpdump — анализ трафика

`tcpdump` переводит сетевой интерфейс в **promiscuous mode** (неразборчивый режим) — карта начинает видеть весь трафик, а не только адресованный её MAC-адресу. tcpdump распознаёт множество протоколов: ARP, RARP, ICMP, TCP, UDP, IP, IPv6 и другие.

### Базовое использование

```bash
# Все пакеты на интерфейсе по умолчанию (нужен root)
sudo tcpdump

# Конкретный интерфейс
sudo tcpdump -i eth0

# Ограничить вывод (первые 10 пакетов)
sudo tcpdump -c 10
```

### Фильтры (примитивы)

tcpdump использует язык фильтров BPF (описан в `man pcap-filter(7)`):

| Примитив | Описание | Пример |
|----------|---------|---------|
| `tcp` | Только TCP | `tcpdump tcp` |
| `udp` | Только UDP | `tcpdump udp` |
| `ip` / `ip6` | IPv4 / IPv6 | `tcpdump ip6` |
| `port N` | TCP/UDP на порту N | `tcpdump port 443` |
| `host H` | Пакеты к/от хоста | `tcpdump host 10.0.1.5` |
| `net N` | Пакеты в/из сети | `tcpdump net 192.168.1.0/24` |
| `src` / `dst` | Направление | `tcpdump src host 10.0.1.5` |

### Комбинирование фильтров

Операторы: `and`, `or`, `!` (not). Группировка — скобки (экранировать в shell). Подробности — в `man pcap-filter(7)`.

```bash
# Веб-пакеты и UDP (пример из книги Уорда)
sudo tcpdump udp or port 80 or port 443

# HTTP или HTTPS трафик
sudo tcpdump port 80 or port 443

# TCP-трафик к конкретному серверу
sudo tcpdump tcp and host 10.0.1.5

# Всё, КРОМЕ SSH (чтобы не видеть свою сессию)
sudo tcpdump not port 22

# DNS-запросы (UDP порт 53)
sudo tcpdump udp port 53

# Сложный фильтр: HTTP от конкретного хоста
sudo tcpdump 'tcp port 80 and src host 192.168.1.100'
```

### Полезные флаги

```bash
# -n — не резолвить DNS (быстрее и чище)
# -v — verbose (больше деталей в заголовках)
# -X — показать содержимое пакета (hex + ASCII)
# -w — записать в файл (для анализа в Wireshark)
# -r — читать из файла

# Запись дампа для анализа
sudo tcpdump -w /tmp/capture.pcap -c 1000 port 443

# Открыть потом в Wireshark
wireshark /tmp/capture.pcap
```

> **Важно:** По умолчанию tcpdump показывает только заголовки TCP (транспортный уровень) и IP (сетевой уровень). С `-X` можно вывести всё содержимое пакета, но большая часть трафика сейчас зашифрована TLS. **Не анализируйте трафик в сетях, которые вам не принадлежат, без разрешения.**

> Для регулярного анализа пакетов удобнее графический интерфейс **Wireshark**.

## netcat — ручное подключение к порту

`netcat` (`nc`) — более гибкая замена `telnet host port`. Позволяет подключаться к TCP/UDP-портам, слушать порты, перенаправлять stdin/stdout в сетевые соединения.

> **Особенность поведения:** по умолчанию netcat выводит минимум отладочной информации. Если что-то не удаётся, он молча завершает работу, устанавливая соответствующий **код выхода**. Добавляйте `-v` для диагностики. При перенаправлении stdin в netcat можно не получить prompt обратно после отправки данных — прервать соединение можно через `Ctrl+C`.

### Подключение к порту

```bash
# Проверить, доступен ли порт (замена telnet)
nc -v server 22
# Connection to server 22 port [tcp/ssh] succeeded!

# Ручной HTTP-запрос (для отладки)
nc server 80
GET / HTTP/1.1
Host: server
# [ответ сервера]

# Таймаут (не зависать бесконечно)
nc -w 3 server 80
```

### Прослушивание порта

```bash
# Слушать порт (сервер-заглушка для тестирования)
nc -l 8080
# Ждёт подключения, выводит всё что получит в stdout

# С другой машины подключиться:
nc server 8080
# Всё что набираете — отправляется на сервер
```

### Передача файлов (без scp/sftp)

```bash
# На принимающей стороне:
nc -l 9999 > received_file.tar.gz

# На отправляющей:
nc server 9999 < file.tar.gz
```

### Сканирование портов (быстрая проверка)

```bash
# Проверить диапазон портов
nc -zv server 20-25
# Connection to server 22 port [tcp/ssh] succeeded!
# nc: connect to server port 20 (tcp) failed: Connection refused
```

Ключевые флаги:

| Флаг | Назначение |
|------|-----------|
| `-v` | Verbose — показывать статус подключения |
| `-l` | Listen mode — слушать порт |
| `-z` | Zero-I/O — только проверить порт, не передавать данные |
| `-w N` | Таймаут N секунд |
| `-u` | UDP вместо TCP |
| `-4` / `-6` | Принудительно IPv4 / IPv6 |

> **IPv4/IPv6 поведение:** по умолчанию клиент netcat пытается подключиться через IPv4 и IPv6. Однако в режиме сервера (`-l`) по умолчанию используется только IPv4. Для IPv6 указывайте `-6` явно.

> **Альтернатива:** если нужно завершить сетевое подключение по окончании стандартного входного потока (а не по `Ctrl+C`), попробуйте утилиту `sock`.

## nmap — сканирование портов извне

`nmap` (Network Mapper) сканирует все порты на хосте или сети в поисках открытых портов. Незаменим для проверки «что видно снаружи» — то, что пропускает firewall.

> **Рекомендация:** запускайте сканирование nmap **как минимум из двух точек** — с самого сервера и с другой машины (желательно за пределами локальной сети). Это покажет, что именно блокирует ваш брандмауэр.

```bash
# Базовое сканирование хоста
nmap 10.1.2.2
# PORT     STATE  SERVICE
# 22/tcp   open   ssh
# 25/tcp   open   smtp
# 80/tcp   open   http
# 111/tcp  open   rpcbind    ← часто единственная включённая по умолчанию
# 8800/tcp open   unknown
# 9090/tcp open   zeus-admin

# Сканировать конкретные порты
nmap -p 22,80,443,8080 10.0.1.5

# Диапазон портов
nmap -p 1-1024 10.0.1.5

# Все 65535 портов (медленно, но полно)
nmap -p- 10.0.1.5

# Определить версии сервисов
nmap -sV 10.0.1.5

# IPv6 — удобный способ определить, какие службы НЕ поддерживают IPv6
nmap -6 server.example.com

# Сканирование подсети (найти живые хосты)
nmap -sn 192.168.1.0/24
```

> **Важно:** Если сеть контролирует кто-то другой — **попросите разрешение**. Сетевые администраторы отслеживают сканирование портов и обычно отключают доступ к машинам, с которых оно выполняется.

### Рекомендуемая проверка: вид изнутри vs вид снаружи

```bash
# На самом сервере — что слушает
ss -tlnp

# С другой машины — что видно через firewall
nmap server-ip

# Разница = то, что блокирует firewall
```

## Сценарии диагностики

### «Приложение не отвечает на порту»

```bash
# 1. Сервис вообще запущен?
systemctl status myapp

# 2. Слушает ли нужный порт?
ss -tlnp | grep 8080
# или
sudo lsof -iTCP:8080 -sTCP:LISTEN

# 3. Если слушает — проблема в сети/firewall?
# С клиента:
nc -zv server 8080

# 4. Firewall блокирует?
sudo iptables -L -n | grep 8080
# или
sudo nft list ruleset | grep 8080

# 5. Пакеты доходят вообще?
sudo tcpdump -i eth0 port 8080 -c 5
```

### «Порт занят, не могу запустить сервис»

```bash
# Кто занимает порт?
sudo lsof -i :8080
# COMMAND  PID  USER  NAME
# node     3456 app   *:8080 (LISTEN)

# Убить процесс
sudo kill 3456

# Или — может это зомби-сокет (TIME_WAIT)?
ss -tn | grep 8080
# TIME-WAIT означает: подождите ~60 сек или используйте SO_REUSEADDR
```

### «Хочу понять, какой трафик идёт к моему серверу»

```bash
# Весь трафик кроме SSH (иначе увидите свою сессию)
sudo tcpdump -n not port 22

# Только HTTP
sudo tcpdump -n port 80 or port 443

# Записать для детального анализа
sudo tcpdump -w /tmp/traffic.pcap -c 5000 not port 22
# Потом открыть в Wireshark
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Забыл `-n` в lsof/tcpdump | Вывод зависает (DNS резолвинг) | Всегда `-n` для быстрого результата |
| `Connection refused` | Порт закрыт (процесс не слушает) | `ss -tlnp` — проверить слушает ли |
| `Connection timed out` | Firewall дропает (не reject) | `nmap` с другой машины / проверить iptables |
| tcpdump показывает свою SSH-сессию | Шум в выводе | `tcpdump not port 22` |
| nmap показывает `filtered` | Firewall отбрасывает пакеты | Проверить iptables/nftables на сервере |
| `Permission denied` при lsof/tcpdump | Нужен root | `sudo` или `capabilities` (`cap_net_raw` для tcpdump) |

## Связанные материалы

- [[linux/how-to/monitor-system]] — мониторинг CPU, RAM, диска, логов
- [[linux/how-to/configure-firewall]] — настройка iptables/nftables
- [[linux/explanation/networking]] — модель сетевых уровней Linux
- [[ssh/explanation/ssh-service-architecture]] — SSH как сетевая служба
