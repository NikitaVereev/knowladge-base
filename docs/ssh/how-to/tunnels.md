---
title: "SSH-туннели"
type: how-to
tags: [ssh, tunnels, port-forwarding, socks, proxy, reverse-tunnel]
related:
  - "[[ssh/how-to/configure-client]]"
  - "[[ssh/reference/cheatsheet]]"
---

# SSH-туннели

> **TL;DR:** `-L` = local forward (достучаться до удалённого сервиса). `-R` = reverse (опубликовать локальный сервис).
> `-D` = SOCKS proxy (весь трафик через сервер).

## Local Port Forwarding (-L)

Доступ к удалённому сервису через локальный порт.

```bash
ssh -L LOCAL_PORT:REMOTE_HOST:REMOTE_PORT user@ssh-server
```

```
Ваш компьютер:8080  ──SSH──►  ssh-server  ──►  db-server:5432
```

### Примеры

```bash
# Доступ к удалённой PostgreSQL через localhost:5432
ssh -L 5432:localhost:5432 user@db-server

# БД на другом хосте за SSH-сервером
ssh -L 5432:10.0.1.50:5432 user@bastion

# Удалённый веб-интерфейс
ssh -L 8080:localhost:3000 user@server
# Открыть http://localhost:8080 → увидите server:3000

# В фоне (без интерактивной сессии)
ssh -fNL 5432:localhost:5432 user@server
# -f = background, -N = no command
```

## Remote Port Forwarding (-R)

Опубликовать локальный сервис через удалённый сервер.

```bash
ssh -R REMOTE_PORT:LOCAL_HOST:LOCAL_PORT user@ssh-server
```

```
Интернет:8080  ──►  ssh-server:8080  ──SSH──►  ваш компьютер:3000
```

### Примеры

```bash
# Показать локальный сайт (localhost:3000) через сервер:8080
ssh -R 8080:localhost:3000 user@server
# Кто-то открывает http://server:8080 → видит ваш localhost:3000

# В фоне
ssh -fNR 8080:localhost:3000 user@server
```

> На сервере в sshd_config нужно: `GatewayPorts yes` (чтобы слушать на 0.0.0.0, не только localhost).

## Dynamic Port Forwarding (-D, SOCKS proxy)

Весь трафик через SSH-сервер (как VPN).

```bash
ssh -D 1080 user@server
```

Настроить браузер: SOCKS5 proxy → `localhost:1080`. Весь трафик браузера пойдёт через `server`.

```bash
# Или curl через proxy
curl --socks5 localhost:1080 http://example.com
```

## В ~/.ssh/config

```
Host db-tunnel
    HostName bastion.example.com
    User admin
    LocalForward 5432 db-internal:5432
    LocalForward 6379 redis-internal:6379

Host web-proxy
    HostName server.example.com
    User deploy
    DynamicForward 1080
```

```bash
ssh db-tunnel     # автоматически пробросит порты
ssh web-proxy     # SOCKS proxy на localhost:1080
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `bind: Address already in use` | Локальный порт занят. Использовать другой или `kill` процесс |
| Reverse tunnel не доступен извне | `GatewayPorts yes` в sshd_config на сервере |
| Туннель отваливается | `ServerAliveInterval 60` в config. Или `autossh` |
| Забыли `-N`, сессия висит | `-N` = no remote command (только туннель) |
