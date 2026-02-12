---
title: "Модель прав доступа"
type: explanation
tags: [linux, permissions, users, groups, chmod, chown, sudo, rwx, acl]
sources:
  original: "_inbox/01-linux/01-core-concepts/02-users-groups-permissions.md"
related:
  - "[[linux/explanation/filesystem]]"
  - "[[linux/reference/permissions-table]]"
  - "[[linux/how-to/manage-users]]"
---

# Модель прав доступа Linux

> **TL;DR:** Каждый файл имеет владельца (user), группу (group) и права для остальных (other).
> Три типа прав: read (r=4), write (w=2), execute (x=1). `chmod 755` = rwxr-xr-x.
> `sudo` — выполнить от root. root (uid=0) может всё.

## Пользователи и группы

Каждый процесс и файл принадлежат пользователю и группе.

**Ключевые файлы:**

| Файл | Содержит | Формат |
|------|---------|--------|
| `/etc/passwd` | Пользователи | `user:x:uid:gid:comment:home:shell` |
| `/etc/shadow` | Хеши паролей | Только root может читать |
| `/etc/group` | Группы | `group:x:gid:member1,member2` |

Специальный пользователь `root` (uid=0) — имеет неограниченные права. Обычные пользователи имеют uid ≥ 1000, системные сервисы — uid 1-999.

```bash
whoami                     # текущий пользователь
id                         # uid, gid, все группы
id username                # информация о другом пользователе
groups                     # мои группы
```

## Права на файлы (rwx)

```bash
ls -la
# -rw-r--r-- 1 user group 1234 Jan 1 12:00 file.txt
# │├──┤├──┤├──┤
# │ u    g    o
# │ ser  roup ther
# │
# тип файла (- = файл, d = директория, l = symlink)
```

| Право | Файл | Директория |
|-------|------|-----------|
| `r` (read, 4) | Читать содержимое | Листать содержимое (`ls`) |
| `w` (write, 2) | Изменять содержимое | Создавать/удалять файлы внутри |
| `x` (execute, 1) | Выполнять как программу | Входить (`cd`) |

## Числовой формат (octal)

Права кодируются тремя цифрами: `user` `group` `other`. Каждая = r+w+x.

| Число | Права | Значение |
|-------|-------|---------|
| 7 | rwx | Полный доступ |
| 6 | rw- | Чтение + запись |
| 5 | r-x | Чтение + выполнение |
| 4 | r-- | Только чтение |
| 0 | --- | Нет доступа |

Типичные комбинации:

| chmod | Значение | Применение |
|-------|---------|-----------|
| `755` | rwxr-xr-x | Директории, скрипты |
| `644` | rw-r--r-- | Обычные файлы |
| `600` | rw------- | Приватные файлы (SSH ключи) |
| `700` | rwx------ | Приватные директории (.ssh/) |
| `777` | rwxrwxrwx | ⚠️ Никогда не используйте в production |

## chmod — изменить права

```bash
# Числовой формат
chmod 755 script.sh
chmod 600 ~/.ssh/id_rsa

# Символьный формат
chmod u+x script.sh        # user: добавить execute
chmod g-w file.txt          # group: убрать write
chmod o= file.txt           # other: убрать всё
chmod a+r file.txt          # all: добавить read

# Рекурсивно
chmod -R 755 /var/www/
```

## chown — изменить владельца

```bash
chown user file.txt                # сменить владельца
chown user:group file.txt          # владелец + группа
chown -R www-data:www-data /var/www/  # рекурсивно
chgrp group file.txt               # только группу
```

## Специальные биты

| Бит | Числ. | Эффект |
|-----|-------|--------|
| **SUID** (Set User ID) | 4xxx | Файл выполняется от имени владельца, а не запустившего. Пример: `/usr/bin/passwd` |
| **SGID** (Set Group ID) | 2xxx | Файл выполняется от имени группы. На директории: новые файлы наследуют группу |
| **Sticky bit** | 1xxx | Удалять файлы из директории может только их владелец. Пример: `/tmp` |

```bash
chmod 4755 program          # SUID
chmod 2755 shared_dir/      # SGID
chmod 1777 /tmp/            # Sticky bit
ls -la /tmp/
# drwxrwxrwt  ← t = sticky bit
```

## sudo и su

```bash
# Выполнить одну команду от root
sudo apt update

# Стать root
sudo -i

# Стать другим пользователем
su - username

# Редактировать sudoers (ТОЛЬКО через visudo!)
sudo visudo
```

Файл `/etc/sudoers`:
```
# user может всё
username ALL=(ALL) ALL

# user без пароля
deploy ALL=(ALL) NOPASSWD:ALL

# группа sudo может всё
%sudo ALL=(ALL) ALL
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| «Permission denied» на скрипте | `chmod +x script.sh` — добавить execute |
| SSH ключ не работает | `chmod 600 ~/.ssh/id_rsa`, `chmod 700 ~/.ssh/` |
| Не могу создать файл в `/var/www` | `chown -R $USER:$USER /var/www/` или используйте `sudo` |
| `chmod 777` как «решение» | ⚠️ Даёт всем полные права. Разберитесь какие права реально нужны |
| Забыли пароль root | Загрузиться в recovery mode, `passwd root` |
