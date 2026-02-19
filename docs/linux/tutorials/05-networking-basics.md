---
title: "05 — Основы сети"
type: tutorial
tags: [linux, tutorial, networking, ip, dns, ssh, firewall, ports, curl]
related:
  - "[[linux/tutorials/04-shell-and-scripting]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/explanation/networking]]"
---

# Tutorial 05 — Основы сети

> **Цель:** Проверить сеть, понять IP/DNS/порты, подключиться по SSH, базовая диагностика.

**Время:** ~25 минут
**Требования:** Терминал Linux с доступом к сети.

## Шаг 1. Проверить подключение

```bash
# Мой IP
ip a                       # все интерфейсы
ip -4 addr show            # только IPv4
hostname -I                # только IP-адреса

# Есть ли интернет?
ping -c 3 8.8.8.8          # по IP (проверяет сеть)
ping -c 3 google.com       # по имени (проверяет сеть + DNS)

# Маршрут по умолчанию (gateway)
ip route
# default via 192.168.1.1 dev eth0
```

## Шаг 2. DNS

```bash
# Резолвить имя → IP
dig google.com +short
nslookup google.com
host google.com

# Локальный DNS
cat /etc/resolv.conf       # текущий DNS-сервер
cat /etc/hosts             # локальные записи (приоритет над DNS)

# Добавить локальную запись
echo "192.168.1.100 myserver" | sudo tee -a /etc/hosts
```

## Шаг 3. Порты и соединения

```bash
# Кто слушает порты
ss -tlnp                   # TCP listening с PID
ss -ulnp                   # UDP
sudo lsof -i :80           # кто на порту 80

# Проверить доступность порта
nc -zv google.com 443      # подключиться к порту
curl -I https://google.com # HTTP-запрос (только заголовки)

# Все соединения
ss -tunp                   # TCP + UDP с PID
```

## Шаг 4. SSH (удалённое подключение)

```bash
# Подключиться
ssh user@192.168.1.100
ssh -p 2222 user@server    # нестандартный порт

# Копировать файлы
scp file.txt user@server:/home/user/
scp -r dir/ user@server:/home/user/
scp user@server:/var/log/syslog ./

# SSH ключи (вместо паролей)
ssh-keygen -t ed25519      # сгенерировать ключ
ssh-copy-id user@server    # скопировать публичный ключ на сервер
# Теперь: ssh user@server — без пароля

# SSH config (~/.ssh/config)
cat > ~/.ssh/config << 'EOF'
Host myserver
    HostName 192.168.1.100
    User admin
    Port 22
    IdentityFile ~/.ssh/id_ed25519
EOF
# Теперь: ssh myserver
```

## Шаг 5. Скачивание и HTTP

```bash
# curl — HTTP запросы
curl https://api.github.com/users/torvalds  # GET
curl -o file.zip https://example.com/file.zip  # скачать
curl -I https://example.com  # только заголовки
curl -X POST -d '{"key":"val"}' -H "Content-Type: application/json" https://api.example.com

# wget — скачивание файлов
wget https://example.com/file.tar.gz
wget -c https://example.com/big-file.iso  # с продолжением (resume)
```

## Шаг 6. Диагностика

```bash
# Трассировка маршрута
traceroute google.com      # через какие узлы идёт пакет
mtr google.com             # интерактивный traceroute

# Проверить интерфейсы
ip link                    # up/down состояние
sudo ip link set eth0 up   # поднять интерфейс

# Перезапустить сеть
sudo systemctl restart NetworkManager
# или
sudo systemctl restart systemd-networkd
```

## Типичные ошибки

| Симптом | Диагностика | Решение |
|---------|------------|---------|
| `ping 8.8.8.8` OK, `ping google.com` не работает | DNS проблема | Проверить `/etc/resolv.conf`, попробовать `nameserver 8.8.8.8` |
| `Connection refused` | Сервис не запущен или firewall | `ss -tlnp | grep PORT`, проверить firewall |
| `No route to host` | Нет маршрута | `ip route`, проверить gateway |
| SSH: `Permission denied` | Неверный ключ/пароль | `ssh -v` для диагностики, проверить `~/.ssh/` права (700/600) |

## Что дальше

→ [[linux/how-to/configure-network]] — статический IP, Wi-Fi, NetworkManager

→ [[linux/how-to/configure-firewall]] — ufw, iptables

→ [[linux/how-to/recipes/ssh-hardening]] — безопасный SSH
