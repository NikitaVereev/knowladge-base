---
title: "Hardening SSH-сервера"
type: how-to
tags: [ssh, security, sshd, hardening, fail2ban, 2fa]
related:
  - "[[ssh/explanation/how-ssh-works]]"
  - "[[ssh/how-to/configure-client]]"
  - "[[linux/how-to/recipes/ssh-hardening]]"
---

# Hardening SSH-сервера

> **TL;DR:** Отключить пароли → только ключи. Сменить порт. fail2ban от brute-force.
> Ограничить пользователей. Отключить root login.

## /etc/ssh/sshd_config

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
sudo vim /etc/ssh/sshd_config
```

### Рекомендуемые настройки

```
# Порт (смена снижает шум от ботов)
Port 2222

# Только ключи (ОТКЛЮЧИТЬ пароли)
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey

# Отключить root login
PermitRootLogin no

# Ограничить пользователей
AllowUsers deploy admin

# Таймауты
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 5

# Отключить ненужное
X11Forwarding no
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# Алгоритмы (только современные)
KexAlgorithms curve25519-sha256@libssh.org
HostKeyAlgorithms ssh-ed25519
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
```

```bash
# Проверить синтаксис
sudo sshd -t

# Применить
sudo systemctl restart sshd
```

> **⚠️ Важно:** Перед перезапуском убедитесь что ваш ключ работает! Держите открытую SSH-сессию на случай ошибки.

## fail2ban

```bash
# Установить
sudo apt install fail2ban    # Ubuntu
sudo pacman -S fail2ban      # Arch

# Конфигурация
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vim /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 2222
filter = sshd
maxretry = 3
bantime = 3600
findtime = 600
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

## Чеклист

```
✅ Ключи вместо паролей (PasswordAuthentication no)
✅ Root login отключён (PermitRootLogin no)
✅ Нестандартный порт (Port 2222+)
✅ fail2ban установлен и работает
✅ AllowUsers ограничивает доступ
✅ MaxAuthTries 3
✅ Firewall: только нужный SSH-порт открыт
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Заблокировали себя | Recovery через консоль хостинга / KVM / IPMI |
| Забыли открыть новый порт в firewall | `ufw allow 2222/tcp` ДО смены порта |
| `PasswordAuthentication no` без ключа на сервере | Сначала `ssh-copy-id`, потом отключать пароли |
| fail2ban заблокировал свой IP | `sudo fail2ban-client set sshd unbanip YOUR_IP` |
