---
title: "Настройка сети"
type: how-to
tags: [linux, networking, ip, networkmanager, nmcli, static-ip, wifi, dns, netplan]
related:
  - "[[linux/tutorials/05-networking-basics]]"
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
  - "[[linux/explanation/networking]]"
---

# Настройка сети

> **TL;DR:** `nmcli` — CLI для NetworkManager (Arch/Fedora/Ubuntu Desktop).
> `netplan` — Ubuntu Server. Статический IP, DNS, Wi-Fi, VLAN.

## Диагностика

```bash
ip a                               # интерфейсы и IP
ip route                           # маршруты (gateway)
cat /etc/resolv.conf               # DNS
ss -tlnp                           # listening ports
ping -c 3 8.8.8.8                  # сеть работает?
ping -c 3 google.com               # DNS работает?
```

## NetworkManager (nmcli)

```bash
# Статус
nmcli general status
nmcli device status
nmcli connection show

# Wi-Fi
nmcli device wifi list
nmcli device wifi connect "SSID" password "PASSWORD"

# Статический IP
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8,8.8.4.4"
nmcli connection up "Wired connection 1"

# Вернуть DHCP
nmcli connection modify "Wired connection 1" ipv4.method auto
nmcli connection up "Wired connection 1"

# Добавить новое подключение
nmcli connection add type ethernet con-name "server" ifname eth0 \
  ipv4.method manual ipv4.addresses 10.0.0.10/24 ipv4.gateway 10.0.0.1
```

## Netplan (Ubuntu Server)

```yaml
# /etc/netplan/01-config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
sudo netplan apply                 # применить
sudo netplan try                   # применить с откатом через 120с
```

## DNS

```bash
# systemd-resolved (Ubuntu)
resolvectl status
sudo resolvectl dns eth0 8.8.8.8 1.1.1.1

# Или вручную
sudo nano /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 1.1.1.1

# /etc/hosts — локальные записи (приоритет над DNS)
echo "192.168.1.50 db.local" | sudo tee -a /etc/hosts
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| IP меняется после reboot | Настроить static IP через nmcli/netplan |
| DNS не работает | Проверить `/etc/resolv.conf`, `resolvectl status` |
| `netplan apply` — ошибка синтаксиса | YAML чувствителен к пробелам. Проверить через `netplan try` |
| NetworkManager vs systemd-networkd конфликт | Использовать только одно. Ubuntu Desktop → NM, Server → networkd |