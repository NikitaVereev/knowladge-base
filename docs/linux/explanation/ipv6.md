---
title: "Автоматическая настройка IPv6"
type: explanation
tags: [linux, networking, ipv6, slaac, dhcpv6, link-local, ra, neighbor-discovery]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.20"
  rfc: "https://datatracker.ietf.org/doc/html/rfc4862"
related:
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/dhcp]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/explanation/arp-ndp]]"
---

# Автоматическая настройка IPv6

> **TL;DR:** IPv6 спроектирован с автоконфигурацией «из коробки». Хост получает link-local адрес (`fe80::`) автоматически, без серверов. Для глобального адреса — SLAAC: маршрутизатор рассылает префикс сети (Router Advertisement), хост дополняет его своим идентификатором. DHCPv6 опционален — нужен только для дополнительных параметров (DNS, NTP) или когда администратор хочет контролировать выдачу адресов.

## Зачем это знать

IPv4-адреса исчерпаны (с 2011 года). IPv6 — не «когда-нибудь», а реальность: крупные облака, мобильные операторы, CDN уже работают по IPv6. Главное отличие от IPv4 в контексте настройки — IPv6 **не нуждается в DHCP** для базовой работы. Хост сам генерирует себе адрес.

## Формат адреса IPv6

128 бит, записывается как восемь групп по 4 hex-цифры:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

Правила сокращения:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334   полный
2001:db8:85a3:0:0:8a2e:370:7334           убрать ведущие нули
2001:db8:85a3::8a2e:370:7334              :: заменяет группы нулей (один раз)
```

### Типы адресов

| Префикс | Тип | Аналог в IPv4 | Описание |
|---------|-----|--------------|----------|
| `::1/128` | Loopback | `127.0.0.1` | Сам хост |
| `fe80::/10` | Link-local | `169.254.x.x` | Только в пределах одного сегмента |
| `2000::/3` | Global Unicast | Публичные IP | Маршрутизируемые в интернете |
| `fc00::/7` (`fd00::/8`) | Unique Local | `10.0.0.0/8`, `192.168.0.0/16` | Частные сети |
| `ff00::/8` | Multicast | `224.0.0.0/4` | Групповая рассылка |

В IPv6 **нет broadcast**. Его функции выполняет multicast (например, `ff02::1` — все узлы на link).

## Три уровня автоконфигурации

### 1. Link-local адрес (всегда, автоматически)

При поднятии интерфейса ядро **автоматически** генерирует link-local адрес `fe80::`. Никаких серверов не нужно.

```
fe80:: + идентификатор интерфейса (из MAC или случайный)
```

```bash
$ ip -6 addr show eth0
inet6 fe80::5054:ff:fe12:3456/64 scope link
#     ↑ link-local, сгенерирован автоматически
```

Link-local достаточен для общения внутри одного сегмента: Neighbor Discovery, обнаружение маршрутизаторов. Но он **не маршрутизируется** — для выхода за пределы сегмента нужен глобальный адрес.

### 2. SLAAC — глобальный адрес без DHCP

SLAAC (Stateless Address Autoconfiguration) — основной механизм автоконфигурации IPv6.

```
Хост                                   Маршрутизатор
  │                                         │
  │  1. Router Solicitation (RS)            │
  │  "Есть тут маршрутизатор?"             │
  │  dst: ff02::2 (all-routers multicast)  │
  │────────────────────────────────────────→│
  │                                         │
  │  2. Router Advertisement (RA)           │
  │  "Префикс сети: 2001:db8:1::/64        │
  │   Мой адрес: fe80::1                   │
  │   Флаги: A=1 (используй SLAAC)        │
  │   Время жизни: 1800с"                  │
  │←────────────────────────────────────────│
  │                                         │
  ▼
Хост генерирует глобальный адрес:
  2001:db8:1:: + собственный идентификатор
  = 2001:db8:1::5054:ff:fe12:3456/64

Хост добавляет default route:
  default via fe80::1 dev eth0
```

Маршрутизатор также периодически рассылает RA (обычно каждые 200–600 секунд) на `ff02::1` (все узлы) — новые хосты не обязаны отправлять RS, достаточно подождать.

**Stateless** означает, что маршрутизатор не хранит информацию о том, какие адреса выданы. Каждый хост самостоятельно генерирует уникальный адрес из префикса + идентификатора.

### 3. DHCPv6 — когда SLAAC недостаточно

SLAAC выдаёт только адрес и gateway. Для DNS, NTP, доменного имени и других параметров нужен DHCPv6 или RDNSS (DNS-информация в RA).

Флаги в Router Advertisement определяют поведение:

| Флаг | Значение | Результат |
|------|----------|----------|
| A=1 (Autonomous) | Использовать SLAAC для адреса | Хост сам генерирует IP |
| M=1 (Managed) | Использовать DHCPv6 для адреса | Хост запрашивает IP у DHCPv6 |
| O=1 (Other) | Использовать DHCPv6 для доп. параметров | SLAAC для IP, DHCPv6 для DNS и т.д. |

Типичная конфигурация: `A=1, O=1` — адрес через SLAAC, DNS через DHCPv6 (или RDNSS в RA).

## Генерация идентификатора интерфейса

64 бита адреса — префикс сети (от маршрутизатора), 64 бита — идентификатор интерфейса (генерирует хост).

| Метод | Описание | Приватность |
|-------|----------|------------|
| EUI-64 (из MAC) | `52:54:00:12:34:56` → `::5054:00ff:fe12:3456` | Плохая: MAC = fingerprint |
| Stable Privacy (RFC 7217) | Хеш от MAC + префикса + секрета | Хорошая: стабильный, но не отслеживаемый |
| Temporary (RFC 8981) | Случайный, меняется со временем | Отличная: для исходящих соединений |

Современные дистрибутивы по умолчанию используют Stable Privacy + Temporary адреса:

```bash
$ ip -6 addr show eth0
inet6 2001:db8:1::a1b2:c3d4:e5f6:7890/64 scope global dynamic mngtmpaddr
#     ↑ stable privacy (основной)
inet6 2001:db8:1::8f2e:7d41:3b19:cafe/64 scope global temporary dynamic
#     ↑ temporary (для исходящих соединений, меняется)
inet6 fe80::5054:ff:fe12:3456/64 scope link
#     ↑ link-local
```

```bash
# Проверить метод генерации
sudo sysctl net.ipv6.conf.eth0.addr_gen_mode
# 0 = EUI-64, 2 = Stable Privacy, 3 = Random

# Включить Privacy Extensions (temporary addresses)
sudo sysctl net.ipv6.conf.eth0.use_tempaddr=2
```

## Neighbor Discovery Protocol (NDP)

NDP заменяет несколько протоколов IPv4: ARP, ICMP Router Discovery, ICMP Redirect. Работает поверх ICMPv6.

| Сообщение | Функция | Аналог IPv4 |
|-----------|---------|-------------|
| Router Solicitation (RS) | Запрос маршрутизатора | — |
| Router Advertisement (RA) | Ответ маршрутизатора с параметрами | — |
| Neighbor Solicitation (NS) | Резолвинг IPv6 → MAC | ARP Request |
| Neighbor Advertisement (NA) | Ответ с MAC-адресом | ARP Reply |
| DAD (Duplicate Address Detection) | Проверка уникальности адреса | Gratuitous ARP |

DAD — перед использованием сгенерированного адреса хост отправляет NS на этот адрес. Если кто-то отвечает — адрес занят, генерируется другой.

## Настройка IPv6 в Linux

```bash
# Проверить IPv6
ip -6 addr show
ip -6 route show

# Отключить IPv6 (если не нужен)
sudo sysctl net.ipv6.conf.all.disable_ipv6=1
sudo sysctl net.ipv6.conf.default.disable_ipv6=1

# Включить обратно
sudo sysctl net.ipv6.conf.all.disable_ipv6=0

# Проверить что IPv6 работает
ping -6 ::1                    # loopback
ping -6 fe80::1%eth0           # link-local (нужен %interface)
curl -6 https://ipv6.google.com
```

Указание интерфейса через `%eth0` обязательно для link-local адресов — они не уникальны глобально (каждый интерфейс имеет свой `fe80::` адрес).

## Dual Stack: IPv4 + IPv6

Большинство современных систем работают в режиме dual stack — оба протокола одновременно. Приложения пытаются сначала IPv6, потом IPv4 (определяется `/etc/gai.conf`).

```bash
$ ip addr show eth0
inet 192.168.1.100/24 ...              # IPv4
inet6 2001:db8:1::a1b2:c3d4/64 ...    # IPv6 global
inet6 fe80::5054:ff:fe12:3456/64 ...   # IPv6 link-local
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Нет глобального IPv6-адреса | Только `fe80::` (link-local) | Маршрутизатор не отправляет RA. Проверить: `rdisc6 eth0`, настроить RA на роутере |
| `connect: Network unreachable` по IPv6 | Нет default route | `ip -6 route` — должен быть `default via fe80::...`. Проблема с RA |
| Медленное подключение к сайтам | Таймаут IPv6, потом fallback на IPv4 | Если IPv6 не работает — отключить: `sysctl net.ipv6.conf.all.disable_ipv6=1` |
| `ping6 fe80::1` — `Invalid argument` | Не указан интерфейс | `ping6 fe80::1%eth0` — для link-local нужен `%iface` |
| Несколько IPv6-адресов на интерфейсе | Это нормально | Link-local + global + temporary — штатное поведение |

## Связанные материалы

- [[linux/explanation/networking]] — стек TCP/IP, IPv4, подсети, маршрутизация
- [[linux/explanation/dhcp]] — DHCPv4, DORA, lease
- [[linux/how-to/configure-network]] — nmcli, netplan, статический IP
