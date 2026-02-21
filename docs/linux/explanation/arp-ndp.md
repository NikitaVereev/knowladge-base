---
title: "Ethernet, IP, ARP и NDP"
type: explanation
tags: [linux, networking, ethernet, arp, ndp, mac, ip, link-layer, neighbor-discovery]
sources:
  docs: "https://man7.org/linux/man-pages/man7/arp.7.html"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.26"
related:
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/ipv6]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/reference/cheatsheet]]"
---

# Ethernet, IP, ARP и NDP

> **TL;DR:** IP-адреса маршрутизируют пакеты между сетями, но внутри одного сегмента (Ethernet) доставка идёт по MAC-адресам. ARP (IPv4) и NDP (IPv6) — протоколы, которые связывают эти два мира: по известному IP узнают MAC соседа. Без них хост знает *куда* отправить пакет, но не знает *кому* физически передать кадр.

## Зачем это знать

- «Сеть не работает» после смены IP, дублирования адреса или переезда VM — часто проблема в устаревшем ARP-кеше
- ARP spoofing — одна из простейших атак в локальной сети (MITM)
- В Docker/Kubernetes: bridge-сети, veth-пары, proxy ARP — всё строится на понимании связи IP ↔ MAC
- NDP в IPv6 заменяет не только ARP, но и DHCP, ICMP Router Discovery — без понимания NDP IPv6-сеть непрозрачна

## Проблема: два уровня адресации

Сетевой стек использует **два независимых** уровня адресов:

| Уровень | Адрес | Область | Назначение |
|---------|-------|---------|-----------|
| Network (L3) | IP-адрес | Глобальная маршрутизация | *Куда* доставить пакет |
| Link (L2) | MAC-адрес | Один сегмент сети | *Кому* физически передать кадр |

Когда хост отправляет пакет соседу в той же подсети, ядро знает IP назначения, но Ethernet-кадр требует **MAC-адрес получателя**. Нужен механизм: IP → MAC.

```
Хост A: 192.168.1.100 (MAC: aa:aa:aa:aa:aa:aa)
Хост B: 192.168.1.200 (MAC: bb:bb:bb:bb:bb:bb)

Хост A хочет отправить пакет на 192.168.1.200.
IP-адрес известен. MAC-адрес — нет.
Вопрос: какой MAC-адрес вписать в Ethernet-кадр?
```

## ARP — Address Resolution Protocol (IPv4)

ARP решает задачу «IP → MAC» для IPv4 через широковещательный запрос в локальном сегменте.

### Как работает ARP

```
1. Хост A: "Кто имеет IP 192.168.1.200? Ответьте на MAC aa:aa:aa:aa:aa:aa"
   → ARP Request (broadcast: ff:ff:ff:ff:ff:ff)
   → Получают ВСЕ хосты в сегменте

2. Хост B (192.168.1.200): "Это я! Мой MAC: bb:bb:bb:bb:bb:bb"
   → ARP Reply (unicast: только хосту A)

3. Хост A: запоминает {192.168.1.200 → bb:bb:bb:bb:bb:bb} в ARP-кеше
   → Отправляет Ethernet-кадр с dst MAC = bb:bb:bb:bb:bb:bb
```

### Структура ARP-пакета

```
Ethernet Header:
  dst: ff:ff:ff:ff:ff:ff (broadcast)     ← все получат
  src: aa:aa:aa:aa:aa:aa                 ← отправитель
  type: 0x0806 (ARP)

ARP Payload:
  operation: 1 (request) / 2 (reply)
  sender MAC: aa:aa:aa:aa:aa:aa
  sender IP:  192.168.1.100
  target MAC: 00:00:00:00:00:00          ← неизвестен (запрос)
  target IP:  192.168.1.200
```

### ARP-кеш

Хост кеширует результаты ARP, чтобы не посылать broadcast на каждый пакет. Записи живут ограниченное время (обычно 30–300 секунд в Linux, зависит от `gc_stale_time`).

```bash
# Просмотр ARP-кеша (таблица соседей)
ip neigh show
# 192.168.1.1 dev eth0 lladdr 52:54:00:12:34:56 REACHABLE
# 192.168.1.200 dev eth0 lladdr bb:bb:bb:bb:bb:bb STALE

# Состояния записей:
# REACHABLE — подтверждён, актуален
# STALE     — устарел, при следующем использовании будет перепроверен
# DELAY     — ожидает перепроверки
# FAILED    — хост не ответил
# PERMANENT — статическая запись (не истекает)
```

```bash
# Очистить ARP-кеш (полезно при отладке)
sudo ip neigh flush dev eth0

# Добавить статическую запись
sudo ip neigh add 192.168.1.200 lladdr bb:bb:bb:bb:bb:bb dev eth0 nud permanent

# Удалить запись
sudo ip neigh del 192.168.1.200 dev eth0
```

### ARP и маршрутизация

Когда пакет идёт **за пределы подсети**, хост не ищет MAC конечного получателя — он ищет MAC **шлюза** (gateway):

```
Хост A (192.168.1.100) → пакет на 8.8.8.8

1. Ядро: 8.8.8.8 не в моей подсети 192.168.1.0/24
   → отправить через gateway 192.168.1.1
2. ARP: "Кто имеет 192.168.1.1?" → получаем MAC шлюза
3. Ethernet-кадр:
   dst MAC: MAC шлюза (52:54:00:12:34:56)  ← L2: соседний хоп
   src MAC: aa:aa:aa:aa:aa:aa
   IP dst: 8.8.8.8                          ← L3: конечный адрес
   IP src: 192.168.1.100
```

Ключевой момент: **IP-адрес не меняется** на всём пути (кроме NAT), а **MAC-адрес перезаписывается на каждом хопе**.

```
Сегмент 1 (192.168.1.0/24):
┌──────────────────────────────────────────┐
│ Ethernet: dst=rr:r0  src=aa:aa           │ ← MAC шлюза
│ IP:       dst=10.0.0.50  src=192.168.1.100│ ← не меняется
└──────────────────────────────────────────┘
       ↓ Router: ARP для 10.0.0.50 на другом интерфейсе
Сегмент 2 (10.0.0.0/24):
┌──────────────────────────────────────────┐
│ Ethernet: dst=ss:ss  src=rr:r1           │ ← новые MAC
│ IP:       dst=10.0.0.50  src=192.168.1.100│ ← те же IP
└──────────────────────────────────────────┘
```

### Gratuitous ARP

«Безвозмездный ARP» — хост отправляет ARP-запрос о **собственном** IP. Не ждёт ответа. Цели:

- **Обновить ARP-кеши** соседей после смены MAC (миграция VM, failover)
- **Обнаружить дубликат IP** — если кто-то ответит, значит адрес уже занят
- **Используется в VRRP/keepalived** — при failover новый master рассылает gratuitous ARP, чтобы трафик пошёл к нему

```bash
# Отправить gratuitous ARP
arping -U -I eth0 192.168.1.100
# -U = unsolicited (gratuitous) ARP
```

## NDP — Neighbor Discovery Protocol (IPv6)

NDP — замена ARP для IPv6, но значительно мощнее. Реализован поверх **ICMPv6** (не как отдельный L2-протокол, в отличие от ARP). Подробнее об автоконфигурации IPv6 через NDP: → [[linux/explanation/ipv6]].

### Что делает NDP (vs ARP)

| Функция | ARP (IPv4) | NDP (IPv6) |
|---------|-----------|-----------|
| IP → MAC | ARP Request/Reply (broadcast) | Neighbor Solicitation/Advertisement (multicast) |
| Обнаружение шлюза | Ручная настройка / DHCP | Router Solicitation/Advertisement (RS/RA) |
| Дубликат IP | Gratuitous ARP (необязательный) | DAD — Duplicate Address Detection (обязательный) |
| Автоконфигурация адреса | Нет (нужен DHCP) | SLAAC |
| Redirect | ICMP Redirect | NDP Redirect |

### Neighbor Solicitation / Advertisement (NS/NA)

Аналог ARP Request/Reply, но вместо broadcast использует **multicast** — эффективнее (получают не все хосты, а только подписанные на solicited-node multicast group).

```
1. Хост A: Neighbor Solicitation
   "Кто имеет fe80::b? Ответьте на мой link-local"
   → ICMPv6 type 135
   → dst: ff02::1:ffXX:XXXX (solicited-node multicast, НЕ broadcast)

2. Хост B: Neighbor Advertisement
   "fe80::b — это я, мой MAC: bb:bb:bb:bb:bb:bb"
   → ICMPv6 type 136
   → unicast к хосту A
```

Solicited-node multicast-адрес формируется из последних 24 бит IPv6-адреса: `ff02::1:ff` + последние 3 байта. NS получат только хосты с похожими суффиксами — не весь сегмент.

### DAD — Duplicate Address Detection

Перед использованием любого IPv6-адреса хост **обязан** проверить его уникальность:

```
1. Хост назначает себе tentative-адрес (ещё не используется)
2. Отправляет NS с src = :: (unspecified) и target = tentative-адрес
3. Ждёт ответ:
   → Никто не ответил → адрес уникален, можно использовать
   → Получил NA → конфликт! Адрес не назначается
```

```bash
# DAD включён по умолчанию
sysctl net.ipv6.conf.eth0.dad_transmits
# 1 = одна проба (по умолчанию)

# Отключить DAD (не рекомендуется, но иногда нужно для контейнеров)
sudo sysctl -w net.ipv6.conf.eth0.accept_dad=0
```

### Таблица соседей в Linux (единая)

Linux хранит и ARP (IPv4), и NDP (IPv6) записи в **единой** таблице соседей:

```bash
# Все записи (IPv4 + IPv6)
ip neigh show
# 192.168.1.1 dev eth0 lladdr 52:54:00:12:34:56 REACHABLE       ← ARP
# fe80::1 dev eth0 lladdr 52:54:00:12:34:56 router REACHABLE     ← NDP

# Только IPv6
ip -6 neigh show

# Мониторинг в реальном времени
ip monitor neigh
```

## Как это работает на практике

### Диагностика

```bash
# 1. Проверить ARP-кеш — знаем ли MAC соседа?
ip neigh show | grep 192.168.1.200
# Нет записи → ARP-запрос не получил ответа

# 2. Послать ARP вручную
arping -I eth0 192.168.1.200
# ARPING 192.168.1.200 from 192.168.1.100 eth0
# Unicast reply from 192.168.1.200 [bb:bb:...]  → хост жив на L2

# 3. Если arping работает, а ping нет → проблема на L3 (firewall, маршрут)

# 4. Обнаружить дубликат IP
arping -D -I eth0 192.168.1.200
# -D = duplicate detection mode
# Получили ответ — адрес дублируется!

# 5. Сниффить ARP-трафик
sudo tcpdump -i eth0 arp
# 12:34:56.789 ARP, Request who-has 192.168.1.200 tell 192.168.1.100
# 12:34:56.790 ARP, Reply 192.168.1.200 is-at bb:bb:bb:bb:bb:bb
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Устаревший ARP-кеш | После миграции VM трафик идёт на старый хост | `ip neigh flush dev eth0` на маршрутизаторе, или gratuitous ARP с нового хоста |
| Дубликат IP в сети | Соединения рвутся, «то работает, то нет» | `arping -D` для обнаружения, убрать дубликат |
| ARP spoofing (MITM) | Атакующий перехватывает трафик в локальной сети | Статические ARP-записи для критичных хостов, Dynamic ARP Inspection (DAI) на свитче |
| ARP table overflow | `Neighbour table overflow` в dmesg | Увеличить `net.ipv4.neigh.default.gc_thresh3` |
| DAD false positive (IPv6) | Адрес не назначается в контейнере | `accept_dad=0` для контейнерного интерфейса |
| Proxy ARP неожиданно включён | Хост отвечает на ARP за чужие IP | Проверить `net.ipv4.conf.*.proxy_arp`, отключить если не нужен |

## Связанные материалы

- [[linux/explanation/networking]] — стек TCP/IP, IP-адреса, подсети, маршрутизация
- [[linux/explanation/ipv6]] — SLAAC, link-local, Router Advertisement (NDP в контексте автоконфигурации)
- [[linux/how-to/configure-network]] — nmcli, netplan, интерфейсы
- [[linux/reference/cheatsheet]] — ip neigh, arping, tcpdump
