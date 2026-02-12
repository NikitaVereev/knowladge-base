---
title: "Рецепт: Первоначальная настройка сервера"
type: how-to
tags: [linux, recipe, server, setup, ssh, firewall, user, timezone]
related:
  - "[[linux/how-to/harden-server]]"
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/recipes/ssh-hardening]]"
---

# Рецепт: Первоначальная настройка сервера

> Выполнить сразу после установки: обновления → пользователь → SSH ключи → firewall → timezone.
> Подходит для Ubuntu Server и Debian. Для Arch — заменить apt на pacman.

## Шаг за шагом

### 1. Подключиться

```bash
ssh root@YOUR_SERVER_IP
```

### 2. Обновить систему

```bash
apt update && apt upgrade -y
apt install -y vim curl wget htop git ufw fail2ban
```

### 3. Создать пользователя

```bash
adduser deploy                     # интерактивно (имя, пароль)
usermod -aG sudo deploy

# Проверить
su - deploy
sudo whoami                        # → root
exit
```

### 4. SSH ключи

```bash
# На ЛОКАЛЬНОЙ машине:
ssh-keygen -t ed25519 -C "server"
ssh-copy-id deploy@YOUR_SERVER_IP

# Проверить вход по ключу
ssh deploy@YOUR_SERVER_IP
```

### 5. Защитить SSH

```bash
sudo vim /etc/ssh/sshd_config
```

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

```bash
sudo systemctl restart sshd
# ⚠️ НЕ закрывайте текущую сессию! Откройте новый терминал и проверьте вход.
```

### 6. Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

### 7. fail2ban

```bash
sudo cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
EOF

sudo systemctl enable --now fail2ban
```

### 8. Timezone и NTP

```bash
sudo timedatectl set-timezone Europe/Tallinn
sudo timedatectl set-ntp true
timedatectl
```

### 9. Hostname

```bash
sudo hostnamectl set-hostname myserver
echo "127.0.1.1 myserver" | sudo tee -a /etc/hosts
```

### 10. Автообновления (Ubuntu)

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Проверочный чеклист

```bash
# Всё ли работает?
ssh deploy@SERVER                  # вход по ключу ✓
sudo ufw status                    # firewall active ✓
sudo systemctl status fail2ban     # fail2ban running ✓
timedatectl                        # timezone correct ✓
sudo systemctl status sshd         # SSH running ✓
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Заблокировали root SSH до создания пользователя | Консоль VPS → исправить sshd_config |
| Забыли `ufw allow 22` перед `enable` | Консоль VPS → `ufw disable` |
| `PasswordAuthentication no` до копирования ключа | Консоль → временно включить пароли |