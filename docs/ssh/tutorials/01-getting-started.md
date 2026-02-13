---
title: "01 — Начало работы с SSH"
type: tutorial
tags: [ssh, tutorial, keys, ed25519, ssh-agent, ssh-copy-id, connection]
sources:
  original: "tools/ssh/how-to/generate-keys.md"
related:
  - "[[ssh/explanation/how-ssh-works]]"
  - "[[ssh/how-to/configure-client]]"
  - "[[ssh/reference/cheatsheet]]"
---

# Tutorial 01 — Начало работы с SSH

> **Цель:** Сгенерировать SSH-ключ Ed25519, скопировать на сервер, подключиться без пароля.
> Настроить ssh-agent для passphrase.

**Время:** ~15 минут
**Требования:** Терминал, доступ к серверу (или VM).

## Шаг 1. Генерация ключа

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

- `Enter` — сохранить в `~/.ssh/id_ed25519` (стандартный путь)
- **Passphrase:** пустой для тестов, с паролем для production

Результат — два файла:
- `~/.ssh/id_ed25519` — **приватный ключ** (НИКОМУ не передавать)
- `~/.ssh/id_ed25519.pub` — **публичный ключ** (копировать на серверы)

## Шаг 2. Права доступа

SSH строго проверяет права. Если неправильные — откажет в подключении.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

## Шаг 3. Копирование ключа на сервер

```bash
# Стандартный порт (22)
ssh-copy-id user@server-ip

# Нестандартный порт
ssh-copy-id -p 2222 user@127.0.0.1
```

Если `ssh-copy-id` недоступен:
```bash
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

На сервере проверить:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## Шаг 4. Подключение

```bash
ssh user@server-ip
# Пароль не запрашивается!

# Нестандартный порт
ssh -p 2222 user@127.0.0.1
```

## Шаг 5. ssh-agent (для passphrase)

Если задали пароль при генерации — agent хранит его в памяти:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
# Введите passphrase один раз

# Проверить загруженные ключи
ssh-add -l
```

## Шаг 6. Выполнение команд удалённо

```bash
# Одна команда
ssh user@server "uptime"

# Несколько команд
ssh user@server "cd /app && git pull && systemctl restart app"

# Копирование файлов (SCP)
scp file.txt user@server:/path/to/
scp -r dir/ user@server:/path/to/

# Копирование файлов (rsync, эффективнее)
rsync -avz local-dir/ user@server:/remote-dir/
```

## Что мы изучили

| Концепция | Что сделали |
|-----------|------------|
| Ed25519 | Современный алгоритм ключей (быстрее и безопаснее RSA) |
| ssh-copy-id | Автоматическое копирование публичного ключа |
| Права доступа | 700 для `.ssh/`, 600 для приватного ключа |
| ssh-agent | Хранит passphrase в памяти до конца сессии |

## Что дальше

→ [[ssh/how-to/configure-client]] — `~/.ssh/config` для удобных подключений
→ [[ssh/how-to/tunnels]] — проброс портов и туннели

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `Permission denied (publickey)` | Проверить права: 600 на ключ, 700 на `.ssh/` (и на сервере тоже) |
| `Too many authentication failures` | Указать ключ явно: `ssh -i ~/.ssh/id_ed25519 user@server` |
| Passphrase спрашивает каждый раз | `eval "$(ssh-agent -s)" && ssh-add` |
| `Host key verification failed` | `ssh-keygen -R server-ip` (сервер был переустановлен) |