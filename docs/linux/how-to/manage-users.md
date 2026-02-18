---
title: "Управление пользователями и группами"
type: how-to
tags: [linux, users, groups, useradd, passwd, sudo, permissions]
related:
  - "[[linux/explanation/permissions-model]]"
  - "[[linux/reference/permissions-table]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
  - "[[linux/explanation/user-files]]"
---

# Управление пользователями и группами

> **TL;DR:** `useradd -m -s /bin/bash user` — создать. `usermod -aG group user` — добавить в группу.
> `passwd user` — сменить пароль. `/etc/passwd` — пользователи, `/etc/group` — группы.

## Пользователи

```bash
# Создать
sudo useradd -m -s /bin/bash -c "John Doe" john
# -m  создать домашнюю директорию
# -s  оболочка
# -c  комментарий (имя)
sudo passwd john                   # задать пароль

# Создать системного пользователя (без home, без login)
sudo useradd -r -s /usr/sbin/nologin appuser

# Изменить
sudo usermod -aG docker john      # добавить в группу (ВАЖНО: -a = append!)
sudo usermod -s /bin/zsh john     # сменить оболочку
sudo usermod -l newname john      # переименовать

# Удалить
sudo userdel john                 # оставить home
sudo userdel -r john              # удалить с home

# Информация
id john                           # uid, gid, groups
getent passwd john                # из /etc/passwd
finger john                       # если установлен
```

## Группы

```bash
# Создать
sudo groupadd developers

# Добавить пользователя
sudo usermod -aG developers john  # ВСЕГДА с -a!
# Без -a → заменит ВСЕ группы на одну

# Удалить из группы
sudo gpasswd -d john developers

# Удалить группу
sudo groupdel developers

# Информация
groups john                       # группы пользователя
getent group developers           # члены группы
```

## sudo

```bash
# Добавить sudo-доступ
sudo usermod -aG sudo john       # Debian/Ubuntu (группа sudo)
sudo usermod -aG wheel john      # Arch/Fedora (группа wheel)

# Или через visudo (безопаснее — проверяет синтаксис)
sudo visudo
# john ALL=(ALL) ALL              — с паролем
# john ALL=(ALL) NOPASSWD:ALL     — без пароля
# john ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx  — конкретная команда

# Проверить
sudo -l -U john                   # что может john
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `usermod -G` без `-a` | Заменяет все группы! Всегда `usermod -aG` |
| Группа не видна после `usermod` | `newgrp group` или перелогиниться |
| `nologin` пользователь не может su | Это штатно — системные пользователи не для логина |
| Забыли пароль root | Recovery mode → `passwd root` |
