---
title: "Справочник: SSH команды"
type: reference
tags: [ssh, reference, cheatsheet, commands, scp, rsync, keygen]
related:
  - "[[ssh/tutorials/01-getting-started]]"
  - "[[ssh/how-to/configure-client]]"
  - "[[ssh/how-to/tunnels]]"
---

# Справочник: SSH команды

## Подключение

```bash
ssh user@host                      # стандартное
ssh -p 2222 user@host              # нестандартный порт
ssh -i ~/.ssh/mykey user@host      # конкретный ключ
ssh -v user@host                   # verbose (отладка)
ssh -vvv user@host                 # максимальный verbose
```

## Ключи

```bash
ssh-keygen -t ed25519 -C "email"   # генерация Ed25519 (рекомендуется)
ssh-keygen -t rsa -b 4096 -C "email" # генерация RSA
ssh-copy-id user@host              # копировать ключ на сервер
ssh-copy-id -p 2222 user@host     # нестандартный порт
ssh-keygen -R host                 # удалить host из known_hosts
ssh-keygen -l -f key.pub           # fingerprint ключа
```

## Agent

```bash
eval "$(ssh-agent -s)"             # запустить agent
ssh-add ~/.ssh/id_ed25519          # добавить ключ
ssh-add -l                         # список загруженных
ssh-add -D                         # удалить все из agent
```

## Копирование файлов

```bash
# SCP
scp file.txt user@host:/path/     # на сервер
scp user@host:/path/file.txt .    # с сервера
scp -r dir/ user@host:/path/      # директорию
scp -P 2222 file user@host:/path/ # нестандартный порт

# rsync (эффективнее)
rsync -avz dir/ user@host:/path/  # синхронизация
rsync -avz --delete src/ dest/    # с удалением лишнего
rsync -avz -e "ssh -p 2222" dir/ user@host:/path/
```

## Туннели

```bash
# Local forward (доступ к удалённому)
ssh -L 8080:localhost:80 user@host
ssh -L 5432:db-server:5432 user@bastion

# Remote forward (опубликовать локальное)
ssh -R 8080:localhost:3000 user@host

# SOCKS proxy
ssh -D 1080 user@host

# В фоне (без интерактивной сессии)
ssh -fNL 5432:localhost:5432 user@host
```

## Удалённое выполнение

```bash
ssh user@host "command"            # одна команда
ssh user@host "cd /app && git pull" # несколько
ssh user@host < script.sh         # выполнить локальный скрипт
ssh -t user@host "sudo command"   # с TTY (для sudo/vim)
```

## Права доступа

| Файл | Права | Описание |
|------|-------|---------|
| `~/.ssh/` | 700 | Директория SSH |
| `~/.ssh/id_ed25519` | 600 | Приватный ключ |
| `~/.ssh/id_ed25519.pub` | 644 | Публичный ключ |
| `~/.ssh/config` | 600 | Конфигурация клиента |
| `~/.ssh/authorized_keys` | 600 | Публичные ключи (сервер) |
| `~/.ssh/known_hosts` | 644 | Fingerprints серверов |

## sshd (сервер)

```bash
sudo sshd -t                      # проверить синтаксис конфига
sudo systemctl restart sshd       # перезапустить
sudo systemctl status sshd        # статус
journalctl -u sshd -f             # логи в реальном времени
```