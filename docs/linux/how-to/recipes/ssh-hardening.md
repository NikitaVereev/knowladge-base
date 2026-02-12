---
title: "Рецепт: SSH Hardening"
type: how-to
tags: [linux, recipe, ssh, security, keys, config, fail2ban, 2fa]
related:
  - "[[linux/how-to/harden-server]]"
  - "[[linux/how-to/configure-firewall]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
---

# Рецепт: SSH Hardening

> Защита SSH: ключи вместо паролей, ограничение пользователей, rate limiting.

## Генерация ключей

```bash
# На ЛОКАЛЬНОЙ машине
ssh-keygen -t ed25519 -C "user@machine"
# → ~/.ssh/id_ed25519 (приватный) — НИКОГДА не передавать
# → ~/.ssh/id_ed25519.pub (публичный) — копировать на серверы

# Скопировать на сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Или вручную на сервере:
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

## Конфигурация sshd

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo vim /etc/ssh/sshd_config
```

```
# Аутентификация
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
MaxAuthTries 3
LoginGraceTime 20

# Ограничить пользователей
AllowUsers deploy admin

# Отключить ненужное
X11Forwarding no
PermitEmptyPasswords no
UsePAM yes

# Таймауты
ClientAliveInterval 300
ClientAliveCountMax 2

# Порт (опционально)
# Port 2222
```

```bash
# Проверить конфиг
sudo sshd -t

# Применить (⚠️ НЕ закрывайте текущую сессию!)
sudo systemctl restart sshd

# Открыть НОВЫЙ терминал и проверить вход
ssh deploy@server
```

## SSH Config (клиент)

```bash
# ~/.ssh/config (на ЛОКАЛЬНОЙ машине)
Host production
    HostName 1.2.3.4
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60

Host staging
    HostName 5.6.7.8
    User deploy
    IdentityFile ~/.ssh/id_ed25519

Host *
    AddKeysToAgent yes
    IdentitiesOnly yes
```

```bash
# Теперь:
ssh production                     # вместо ssh deploy@1.2.3.4
scp file.txt production:/tmp/
```

## fail2ban для SSH

```bash
sudo apt install fail2ban

cat << 'EOF' | sudo tee /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 22
maxretry = 3
findtime = 600
bantime = 3600
banaction = ufw
EOF

sudo systemctl restart fail2ban

# Мониторинг
sudo fail2ban-client status sshd
sudo fail2ban-client get sshd banned
# Разбанить: sudo fail2ban-client set sshd unbanip 1.2.3.4
```

## Права на файлы

```bash
# Сервер
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/id_*.pub
chmod 644 ~/.ssh/config
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| «Permission denied (publickey)» | Проверить права: 700 на .ssh/, 600 на authorized_keys |
| Заблокировали себя | Консоль VPS → `PasswordAuthentication yes` → исправить |
| Ключ не принимается | `ssh -v user@server` — verbose покажет причину |
| fail2ban заблокировал | `fail2ban-client set sshd unbanip YOUR_IP` |