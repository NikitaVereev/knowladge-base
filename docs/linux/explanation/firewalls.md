---
title: "Брандмауэры и netfilter"
type: explanation
tags: [linux, networking, firewall, netfilter, iptables, nftables, packet-filtering, security]
sources:
  docs: "https://www.netfilter.org/documentation/"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.25"
related:
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/linux-router]]"
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/nat]]"
  - "[[linux/how-to/harden-server]]"
---

# Брандмауэры и netfilter

> **TL;DR:** Брандмауэр в Linux — это netfilter, подсистема ядра, которая перехватывает пакеты в определённых точках (hooks) и применяет к ним правила: пропустить, отбросить, изменить. Управляется через iptables (legacy) или nftables (современный). Ключевая идея — цепочки правил на пути пакета через ядро: PREROUTING → FORWARD/INPUT → OUTPUT → POSTROUTING.

## Зачем это знать

Firewall — первая линия защиты сервера. Но он не «ставится и забывается»:

- Неправильный порядок правил — трафик проходит мимо фильтра
- Непонимание пути пакета — NAT-правила не срабатывают, port forwarding не работает
- Docker, Kubernetes, libvirt добавляют собственные цепочки в iptables — конфликты ломают сеть
- Миграция с iptables на nftables требует понимания обеих моделей

## Ключевые концепции

### Netfilter — подсистема ядра

Netfilter — фреймворк в ядре Linux для перехвата и обработки сетевых пакетов. Он определяет **5 точек перехвата** (hooks) на пути пакета через сетевой стек:

```
                              Пакет прибыл
                                  │
                                  ▼
                          ┌──────────────┐
                          │  PREROUTING  │ ← DNAT, connection tracking
                          └──────┬───────┘
                                 │
                          Решение маршрутизации:
                          пакет для нас или транзитный?
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
            ┌──────────┐              ┌──────────┐
            │  INPUT   │              │ FORWARD  │ ← фильтрация транзитного трафика
            └────┬─────┘              └────┬─────┘
                 │                         │
                 ▼                         │
          Локальный процесс                │
          (nginx, sshd, etc.)              │
                 │                         │
                 ▼                         │
            ┌──────────┐                   │
            │  OUTPUT  │                   │
            └────┬─────┘                   │
                 │                         │
                 └────────────┬────────────┘
                              │
                              ▼
                      ┌───────────────┐
                      │ POSTROUTING   │ ← SNAT, masquerade
                      └───────┬───────┘
                              │
                              ▼
                        Пакет уходит
```

- **PREROUTING** — сразу после получения пакета, до решения о маршрутизации. Здесь DNAT (port forwarding) — меняем dst до того, как ядро решит куда направить пакет
- **INPUT** — пакет предназначен локальному процессу на этом хосте
- **FORWARD** — транзитный пакет, проходящий сквозь хост (маршрутизатор)
- **OUTPUT** — пакет сгенерирован локальным процессом
- **POSTROUTING** — перед отправкой в сеть. Здесь SNAT/masquerade — подменяем src после маршрутизации

### Таблицы iptables

iptables группирует правила в **таблицы** по назначению. Каждая таблица содержит **цепочки** (chains), а каждая цепочка — упорядоченный список правил.

| Таблица | Назначение | Цепочки | Когда нужна |
|---------|-----------|---------|-------------|
| **filter** | Фильтрация (пропустить/отбросить) | INPUT, FORWARD, OUTPUT | Всегда — основная таблица |
| **nat** | Подмена адресов (NAT) | PREROUTING, OUTPUT, POSTROUTING | Маршрутизатор, port forwarding |
| **mangle** | Модификация заголовков (TTL, TOS, mark) | Все 5 | Редко — QoS, policy routing |
| **raw** | Обход connection tracking | PREROUTING, OUTPUT | Редко — высоконагруженные серверы |

По умолчанию работает таблица `filter` (если не указать `-t`):

```bash
# Эквивалентные команды:
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT
```

### Правила и действия (targets)

Каждое правило состоит из **условий совпадения** (match) и **действия** (target):

```bash
# Структура правила:
# iptables -t ТАБЛИЦА -A ЦЕПОЧКА [условия] -j ДЕЙСТВИЕ

# Пример: разрешить HTTPS только из внутренней сети
sudo iptables -A INPUT \
  -p tcp \                    # протокол TCP
  --dport 443 \               # destination port 443
  -s 10.0.0.0/8 \             # source IP из подсети 10.0.0.0/8
  -i eth0 \                   # входящий интерфейс eth0
  -m state --state NEW \      # новое соединение (connection tracking)
  -j ACCEPT                   # действие: пропустить
```

| Действие | Описание |
|----------|----------|
| `ACCEPT` | Пропустить пакет |
| `DROP` | Отбросить молча (отправитель не узнает) |
| `REJECT` | Отбросить с ICMP-ответом (отправитель получит ошибку) |
| `LOG` | Залогировать и продолжить обработку (не терминальное) |
| `DNAT` | Изменить destination (таблица nat) |
| `SNAT` | Изменить source (таблица nat) |
| `MASQUERADE` | SNAT с автоопределением IP (таблица nat) |

> **DROP vs REJECT:** `DROP` безопаснее — сканер портов не получает подтверждения. `REJECT` вежливее — клиент сразу получает ошибку, не ждёт таймаут. Для внешнего трафика обычно DROP, для внутреннего — REJECT.

### Порядок имеет значение

Правила проверяются **сверху вниз**, первое совпавшее — срабатывает:

```bash
# ❌ Неправильно: ACCEPT до DROP — запрет не сработает
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # правило 1
sudo iptables -A INPUT -s 10.0.0.5 -p tcp --dport 22 -j DROP  # никогда не сработает!

# ✅ Правильно: специфичные правила первыми
sudo iptables -A INPUT -s 10.0.0.5 -p tcp --dport 22 -j DROP  # конкретный запрет
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # общий доступ
```

### Политика по умолчанию

Если ни одно правило не совпало, применяется **политика цепочки** (policy):

```bash
# Стратегия «default deny» — запретить всё, разрешить явно
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT    # исходящий обычно разрешают

# Обязательно перед этим:
sudo iptables -A INPUT -i lo -j ACCEPT                          # loopback
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # ответы
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT              # SSH
```

> **Важно:** `iptables -P INPUT DROP` без правила для SSH = потеря удалённого доступа. Всегда добавляй allow-правила **до** смены политики.

### Connection Tracking (stateful firewall)

Netfilter отслеживает состояние каждого соединения (модуль `conntrack`). Это позволяет не прописывать отдельные правила для ответного трафика:

| Состояние | Описание |
|-----------|----------|
| `NEW` | Первый пакет нового соединения |
| `ESTABLISHED` | Пакет принадлежит уже установленному соединению |
| `RELATED` | Связан с существующим (ICMP error, FTP data) |
| `INVALID` | Не относится ни к одному известному соединению |

```bash
# Одно правило вместо десятков: разрешить все ответы на наши запросы
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Теперь достаточно разрешать только NEW-соединения для входящего трафика
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
```

Без connection tracking (stateless firewall) пришлось бы явно разрешать входящие пакеты на ephemeral-порты (49152–65535) для ответного трафика — дыра в безопасности.

```bash
# Текущая таблица conntrack
sudo conntrack -L
# tcp  6 300 ESTABLISHED src=192.168.1.100 dst=93.184.216.34 sport=49832 dport=443 ...

# Статистика
sudo conntrack -S
cat /proc/sys/net/netfilter/nf_conntrack_count   # текущих записей
cat /proc/sys/net/netfilter/nf_conntrack_max     # максимум
```

## nftables — замена iptables

nftables — новый фреймворк (с ядра 3.13, 2014), заменяющий iptables/ip6tables/arptables/ebtables единым инструментом `nft`. Debian 10+, RHEL 8+ используют nftables по умолчанию.

### Отличия от iptables

| | iptables | nftables |
|---|---------|---------|
| Утилиты | `iptables`, `ip6tables`, `arptables`, `ebtables` | Одна: `nft` |
| Таблицы | Предопределённые (filter, nat, mangle) | Создаёшь сам |
| Производительность | Линейный обход правил | Оптимизации: sets, maps, concatenations |
| Атомарность | Правила добавляются по одному | Весь ruleset — атомарно (`nft -f`) |
| Синтаксис | Флаги CLI (`-A INPUT -p tcp --dport 22 -j ACCEPT`) | Ближе к языку (`tcp dport 22 accept`) |

### Базовый firewall на nftables

```bash
#!/usr/sbin/nft -f
# /etc/nftables.conf — базовый серверный firewall

flush ruleset                          # очистить всё

table inet filter {                    # inet = IPv4 + IPv6
    chain input {
        type filter hook input priority 0; policy drop;  # default deny

        iifname "lo" accept            # loopback
        ct state established,related accept  # ответы на наши запросы
        ct state invalid drop          # битые пакеты

        # ICMP (ping, MTU discovery, etc.)
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # Разрешённые сервисы
        tcp dport 22 accept            # SSH
        tcp dport { 80, 443 } accept   # HTTP(S)

        # Логировать отброшенное (rate limit чтобы не забить лог)
        limit rate 5/minute log prefix "nft-drop: " drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;  # не маршрутизатор
    }

    chain output {
        type filter hook output priority 0; policy accept;  # исходящее разрешено
    }
}
```

```bash
# Применить конфигурацию
sudo nft -f /etc/nftables.conf

# Просмотреть текущие правила
sudo nft list ruleset

# Добавить правило на лету
sudo nft add rule inet filter input tcp dport 8080 accept

# Удалить правило (по handle)
sudo nft -a list chain inet filter input   # показать handles
sudo nft delete rule inet filter input handle 15

# Включить при загрузке
sudo systemctl enable nftables
```

## Стратегии фильтрации

### Default deny (whitelist)

Запретить всё по умолчанию, разрешить только нужное. **Рекомендуемый подход** — любой пропущенный порт закрыт:

```
policy DROP → явно ACCEPT нужные порты
```

### Default allow (blacklist)

Разрешить всё по умолчанию, запрещать конкретное. Опасно — забытый open-порт = уязвимость:

```
policy ACCEPT → явно DROP опасные порты
```

### Зонирование

Разные правила для разных интерфейсов:

```bash
# nftables: внешний vs внутренний интерфейс
chain input {
    type filter hook input priority 0; policy drop;

    ct state established,related accept

    # Внутренняя сеть — доверенная
    iifname "eth1" accept

    # Внешний интерфейс — только SSH и HTTPS
    iifname "eth0" tcp dport { 22, 443 } accept
}
```

## Взаимодействие с Docker и Kubernetes

Docker и Kubernetes создают собственные цепочки в iptables. Это частый источник проблем.

### Docker

Docker вставляет правила в цепочку `DOCKER` (nat + filter), обходя цепочку `INPUT`. Результат: `ufw deny 8080` не блокирует Docker-порт, потому что трафик идёт через `FORWARD → DOCKER`, минуя `INPUT`.

```bash
# Docker добавляет при -p 8080:80:
sudo iptables -t nat -L DOCKER
# DNAT  tcp  --  anywhere  anywhere  tcp dpt:8080 to:172.17.0.2:80

sudo iptables -L DOCKER
# ACCEPT  tcp  --  anywhere  172.17.0.2  tcp dpt:80
```

Решения:
- Привязывать порты к localhost: `-p 127.0.0.1:8080:80`
- Использовать `DOCKER-USER` цепочку для своих правил (Docker её не перезаписывает)
- `"iptables": false` в `/etc/docker/daemon.json` — ручное управление

### Kubernetes

kube-proxy создаёт цепочки `KUBE-SERVICES`, `KUBE-SVC-*`, `KUBE-SEP-*` для маршрутизации к Pod'ам через DNAT. NetworkPolicy реализуется CNI-плагином (Calico, Cilium) через дополнительные правила в iptables или eBPF.

## Подводные камни

> **Важно:** Никогда не применяй `iptables -P INPUT DROP` через SSH без правила allow для порта 22. Второго шанса не будет (только console/IPMI/KVM).

| Проблема | Симптом | Решение |
|----------|---------|---------|
| Правило не в том порядке | Трафик проходит/блокируется вопреки ожиданиям | `iptables -L -n --line-numbers`, проверить порядок |
| Docker обходит ufw | Порт доступен снаружи несмотря на `ufw deny` | `DOCKER-USER` цепочка или bind на 127.0.0.1 |
| `conntrack table full` | `dropping packet` в dmesg, потеря соединений | Увеличить `nf_conntrack_max`, уменьшить таймауты |
| iptables vs nftables конфликт | Правила «исчезают» или дублируются | Выбрать один инструмент. `update-alternatives --config iptables` |
| REJECT на внешнем интерфейсе | Сканеру видны закрытые порты (получает RST) | Использовать DROP для внешнего трафика |
| Правила теряются после reboot | Firewall «сбрасывается» | `netfilter-persistent save` или `systemctl enable nftables` |

## Связанные материалы

- [[linux/how-to/configure-firewall]] — практика: ufw, iptables, nftables, firewalld
- [[linux/how-to/linux-router]] — ip_forward, masquerade, FORWARD-правила
- [[linux/explanation/networking]] — стек TCP/IP, IP, подсети, маршрутизация
- [[linux/explanation/nat]] — NAT/SNAT/DNAT, masquerade, conntrack
- [[linux/how-to/harden-server]] — SSH, fail2ban, комплексная защита
