---
title: "Настройка Firewall"
type: how-to
tags: [linux, firewall, ufw, iptables, nftables, security, ports]
sources:
  docs: "https://www.netfilter.org/documentation/"
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 9.25"
related:
  - "[[linux/explanation/firewalls]]"
  - "[[linux/how-to/configure-network]]"
  - "[[linux/how-to/recipes/ssh-hardening]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
  - "[[linux/explanation/networking]]"
  - "[[linux/explanation/nat]]"
  - "[[linux/how-to/linux-router]]"
---

# Настройка Firewall

> **TL;DR:** `ufw` — простой firewall (Ubuntu). `ufw allow 22` → `ufw enable`.
> Правило: запретить всё входящее, разрешить только нужные порты.

## ufw (Uncomplicated Firewall)

Обёртка над iptables. Рекомендуется для Ubuntu/Debian.

```bash
# Установка (обычно предустановлен)
sudo apt install ufw

# Статус
sudo ufw status verbose

# Политика по умолчанию
sudo ufw default deny incoming     # запретить все входящие
sudo ufw default allow outgoing    # разрешить все исходящие

# Разрешить порты
sudo ufw allow 22                  # SSH
sudo ufw allow 80                  # HTTP
sudo ufw allow 443                 # HTTPS
sudo ufw allow 22/tcp              # только TCP
sudo ufw allow 60000:61000/udp     # диапазон (mosh)

# По имени сервиса
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'       # 80 + 443
sudo ufw app list                  # список известных приложений

# Разрешить с конкретного IP
sudo ufw allow from 192.168.1.0/24 to any port 22

# Запретить
sudo ufw deny from 10.0.0.5

# Удалить правило
sudo ufw delete allow 80
sudo ufw status numbered           # с номерами
sudo ufw delete 3                  # по номеру

# Включить (⚠️ убедитесь что SSH разрешён!)
sudo ufw enable

# Выключить / Сбросить
sudo ufw disable
sudo ufw reset                     # удалить все правила
```

## iptables (продвинутый)

```bash
# Посмотреть правила
sudo iptables -L -n -v

# Разрешить SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Разрешить established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Разрешить localhost
sudo iptables -A INPUT -i lo -j ACCEPT

# Запретить всё остальное
sudo iptables -A INPUT -j DROP

# Сохранить (Ubuntu)
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

## firewalld (Fedora/RHEL)

```bash
# Статус
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# Разрешить сервис
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp

# Применить
sudo firewall-cmd --reload
```

## Минимальный набор для сервера

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp         # SSH
sudo ufw allow 80/tcp         # HTTP
sudo ufw allow 443/tcp        # HTTPS
sudo ufw enable
sudo ufw status
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Заблокировали себя (SSH) | Физический доступ / консоль VPS → `ufw disable` |
| `ufw enable` без `allow 22` | Потеря SSH. **Всегда** сначала `allow 22` |
| Правила не сохраняются (iptables) | `netfilter-persistent save` или `iptables-save` |
| ufw + Docker конфликт | Docker обходит ufw. Использовать `DOCKER_IPTABLES=false` или UFW-Docker |
