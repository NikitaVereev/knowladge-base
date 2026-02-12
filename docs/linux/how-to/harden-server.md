---
title: "Защита сервера (Hardening)"
type: how-to
tags: [linux, security, hardening, ssh, firewall, updates, fail2ban, audit]
related:
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/manage-users]]"
  - "[[linux/how-to/recipes/ssh-hardening]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
---

# Защита сервера (Hardening)

> **TL;DR:** SSH по ключам + отключить root login + firewall + fail2ban + автообновления.
> Принцип минимальных привилегий. Обновляйте регулярно.

## Чеклист

### 1. Обновления

```bash
# Обновить всё
sudo apt update && sudo apt upgrade    # Ubuntu
sudo pacman -Syu                       # Arch

# Автоматические security-обновления (Ubuntu)
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### 2. SSH

```bash
# /etc/ssh/sshd_config
PermitRootLogin no                     # запретить root SSH
PasswordAuthentication no              # только ключи
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers deploy admin                # только конкретные пользователи
Port 2222                              # нестандартный порт (опционально)
```

```bash
sudo systemctl restart sshd
```

Подробнее: [[linux/how-to/recipes/ssh-hardening]].

### 3. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp                 # или 2222/tcp если сменили
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 4. fail2ban (защита от brute-force)

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
cat << 'EOF' | sudo tee /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 22
EOF

sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd       # статистика
```

### 5. Минимум сервисов

```bash
# Что запущено?
systemctl list-units --type=service --state=running

# Отключить ненужное
sudo systemctl disable --now cups      # принтеры (если не нужны)
sudo systemctl disable --now avahi-daemon  # mDNS
sudo systemctl disable --now bluetooth
```

### 6. Пользователи

```bash
# Создать деплой-пользователя
sudo useradd -m -s /bin/bash -G sudo deploy
sudo passwd deploy

# Заблокировать root пароль (sudo вместо su)
sudo passwd -l root

# Проверить пустые пароли
sudo awk -F: '($2 == "") {print $1}' /etc/shadow
```

### 7. Права на файлы

```bash
# Критичные файлы
sudo chmod 600 /etc/shadow
sudo chmod 644 /etc/passwd
sudo chmod 700 /root
sudo chmod 700 /home/*/.ssh
sudo chmod 600 /home/*/.ssh/authorized_keys
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Заблокировали себя SSH | Консоль VPS / recovery → исправить sshd_config |
| fail2ban заблокировал мой IP | `sudo fail2ban-client set sshd unbanip YOUR_IP` |
| `PasswordAuthentication no` до копирования ключа | Сначала `ssh-copy-id`, потом отключать пароли |
| Забыли `ufw allow ssh` перед `enable` | Консоль VPS → `ufw disable` |