---
title: "Настройка Linux как маршрутизатора"
type: how-to
tags: [linux, networking, router, forwarding, iptables, nftables, nat, masquerade]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.21"
  docs: "https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt"
related:
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/nat]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/how-to/configure-firewall]]"
---

# Настройка Linux как маршрутизатора

> **TL;DR:** Linux может маршрутизировать пакеты между сетями — достаточно включить `ip_forward` и настроить NAT (masquerade). Один параметр ядра (`net.ipv4.ip_forward=1`) превращает хост из конечной точки в маршрутизатор.

## Предварительные условия

- Linux-хост с двумя сетевыми интерфейсами (или интерфейс + VPN)
- root-доступ
- Понимание подсетей и маршрутизации (→ [[linux/explanation/networking]])

## Сценарий

```
Внутренняя сеть              Linux-маршрутизатор              Интернет
192.168.1.0/24               eth1: 192.168.1.1 (внутр.)
                             eth0: получает IP от ISP (внешн.)
Хост A: 192.168.1.100 ──────┘                    └──────── ISP gateway
Хост B: 192.168.1.101 ──────┘
```

Задача: хосты A и B должны выходить в интернет через Linux-маршрутизатор.

## Шаг 1. Включить IP forwarding

По умолчанию ядро Linux **отбрасывает** пакеты, предназначенные не для его IP-адресов. Для маршрутизации это нужно изменить.

```bash
# Проверить текущее состояние
cat /proc/sys/net/ipv4/ip_forward
# 0 = отключено (обычный хост)
# 1 = включено (маршрутизатор)

# Включить (временно, до reboot)
sudo sysctl -w net.ipv4.ip_forward=1

# Включить постоянно
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl --system
```

Для IPv6:

```bash
sudo sysctl -w net.ipv6.conf.all.forwarding=1
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/99-router.conf
```

> **Важно:** Без `ip_forward=1` ядро просто дропает транзитные пакеты. Это самая частая причина «всё настроил, но не работает».

## Шаг 2. Настроить NAT (masquerade)

Внутренние хосты имеют частные адреса (`192.168.1.x`), которые не маршрутизируются в интернете. NAT подменяет source IP на адрес внешнего интерфейса.

### iptables

```bash
# Masquerade: подменять source IP для пакетов из внутренней сети
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# Разрешить forwarding для установленных соединений
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Разрешить forwarding из внутренней сети наружу
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
```

### nftables

```bash
sudo nft add table inet filter
sudo nft add chain inet filter forward { type filter hook forward priority 0 \; policy drop \; }
sudo nft add rule inet filter forward iifname "eth1" oifname "eth0" accept
sudo nft add rule inet filter forward iifname "eth0" oifname "eth1" ct state related,established accept

sudo nft add table nat
sudo nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule nat postrouting oifname "eth0" ip saddr 192.168.1.0/24 masquerade
```

## Шаг 3. Настроить клиентов

Внутренние хосты должны знать, что их gateway — Linux-маршрутизатор.

### Вариант A: DHCP (автоматически)

Установить DHCP-сервер на маршрутизаторе, который выдаёт `gateway=192.168.1.1`.

```bash
# Установить dnsmasq (лёгкий DHCP + DNS)
sudo apt install dnsmasq    # или pacman -S dnsmasq

# /etc/dnsmasq.conf
interface=eth1
dhcp-range=192.168.1.100,192.168.1.200,24h
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,8.8.8.8,8.8.4.4

sudo systemctl enable --now dnsmasq
```

### Вариант B: статически на каждом хосте

```bash
# На хосте A
sudo ip route add default via 192.168.1.1
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

## Шаг 4. Сохранить правила

iptables/nftables правила **теряются** при перезагрузке. Нужно сохранить.

```bash
# iptables: сохранить и восстановить
sudo apt install iptables-persistent         # Debian/Ubuntu
sudo netfilter-persistent save

# Или вручную
sudo iptables-save > /etc/iptables/rules.v4
# Восстановление при загрузке — через systemd unit или /etc/rc.local

# nftables: правила в конфиге
sudo nft list ruleset > /etc/nftables.conf
sudo systemctl enable nftables
```

## Проверка

```bash
# На маршрутизаторе: forwarding включён?
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1

# На маршрутизаторе: NAT-правила есть?
sudo iptables -t nat -L -n
# Chain POSTROUTING
# MASQUERADE  all  --  192.168.1.0/24  0.0.0.0/0

# На клиенте: gateway настроен?
ip route
# default via 192.168.1.1 dev eth0

# На клиенте: интернет работает?
ping -c 2 8.8.8.8
ping -c 2 google.com

# На маршрутизаторе: видны ли транзитные пакеты?
sudo conntrack -L | head
```

## Полная конфигурация (сводка)

```bash
#!/bin/bash
# setup-router.sh — превратить Linux в маршрутизатор
set -euo pipefail

# 1. Forwarding
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-router.conf
sysctl --system

# 2. NAT (iptables)
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# 3. Сохранить
iptables-save > /etc/iptables/rules.v4

echo "Router configured. Internal network: eth1 (192.168.1.0/24), External: eth0"
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| `ip_forward=0` | Клиенты не получают ответов | `sysctl -w net.ipv4.ip_forward=1` |
| Нет MASQUERADE-правила | `ping 8.8.8.8` с клиента — без ответа | `iptables -t nat -L` — проверить наличие правила |
| FORWARD policy DROP, нет allow-правил | Пакеты дропаются на маршрутизаторе | Добавить FORWARD-правила или `iptables -P FORWARD ACCEPT` (менее безопасно) |
| Правила потеряны после reboot | Работало, после перезагрузки — нет | `iptables-persistent` или `nftables.conf` + `systemctl enable` |
| DNS не работает на клиентах | `ping 8.8.8.8` OK, `ping google.com` нет | Настроить DNS на клиентах: `/etc/resolv.conf` или через DHCP-option |
| Асимметричная маршрутизация | Пакеты уходят через один путь, возвращаются через другой | `rp_filter` (reverse path filtering) блокирует. `sysctl net.ipv4.conf.all.rp_filter=0` для отключения |

## Связанные материалы

- [[linux/explanation/networking]] — стек TCP/IP, IP, подсети, маршрутизация
- [[linux/explanation/nat]] — частные сети, NAT/SNAT/DNAT, masquerade, port forwarding
- [[linux/how-to/configure-network]] — nmcli, netplan, интерфейсы
- [[linux/how-to/configure-firewall]] — ufw, iptables/nftables
