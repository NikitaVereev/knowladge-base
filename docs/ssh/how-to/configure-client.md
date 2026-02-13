---
title: "Настройка SSH-клиента"
type: how-to
tags: [ssh, config, client, alias, proxy, jump, multiplexing]
related:
  - "[[ssh/tutorials/01-getting-started]]"
  - "[[ssh/how-to/tunnels]]"
  - "[[ssh/how-to/recipes/ssh-config-examples]]"
---

# Настройка SSH-клиента

> **TL;DR:** `~/.ssh/config` — алиасы для серверов. Вместо `ssh -p 2222 -i ~/.ssh/key user@192.168.1.50`
> просто `ssh myserver`. Поддерживает jump hosts, multiplexing, proxy.

## ~/.ssh/config

```bash
# Создать если не существует
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```

### Базовый пример

```
Host myserver
    HostName 192.168.1.50
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

```bash
# Теперь вместо:
ssh -p 2222 -i ~/.ssh/id_ed25519 deploy@192.168.1.50

# Просто:
ssh myserver
```

### Несколько серверов

```
Host prod
    HostName prod.example.com
    User deploy
    IdentityFile ~/.ssh/prod_key

Host staging
    HostName staging.example.com
    User deploy
    IdentityFile ~/.ssh/staging_key

Host dev
    HostName 192.168.56.10
    User vagrant
    IdentityFile ~/.vagrant/machines/default/virtualbox/private_key
    StrictHostKeyChecking no

# Общие настройки для всех
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

### Wildcard и паттерны

```
# Все серверы в домене
Host *.example.com
    User deploy
    IdentityFile ~/.ssh/work_key

# Все серверы начинающиеся с "node"
Host node*
    User admin
    Port 22
```

## Jump Host (бастион)

```
# Через бастион к внутреннему серверу
Host bastion
    HostName bastion.example.com
    User admin

Host internal
    HostName 10.0.1.50
    User deploy
    ProxyJump bastion
```

```bash
ssh internal
# Автоматически: ssh bastion → ssh 10.0.1.50
```

## Connection Multiplexing

Один TCP-коннект для нескольких SSH-сессий (быстрее):

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
```

```bash
mkdir -p ~/.ssh/sockets
```

## Полезные опции

| Опция | Описание |
|-------|---------|
| `ServerAliveInterval 60` | Ping каждые 60 сек (не рвать при idle) |
| `ServerAliveCountMax 3` | Отключиться после 3 неудачных ping |
| `StrictHostKeyChecking no` | Не спрашивать о новых хостах (для тестовых VM) |
| `AddKeysToAgent yes` | Автоматически добавлять ключи в agent |
| `ForwardAgent yes` | Пробросить agent на сервер (⚠️ только для доверенных) |
| `Compression yes` | Сжатие (полезно на медленных каналах) |

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Config не работает | `chmod 600 ~/.ssh/config` — права обязательны |
| Несколько ключей — используется не тот | `IdentityFile` явно + `IdentitiesOnly yes` |
| `ForwardAgent yes` на чужом сервере | Злоумышленник на сервере получит доступ к вашему agent. Только на своих! |
| Multiplexing socket stale | `rm ~/.ssh/sockets/*` и переподключиться |