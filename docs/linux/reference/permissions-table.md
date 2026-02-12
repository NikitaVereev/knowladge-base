---
title: "Справочник: Права доступа"
type: reference
tags: [linux, permissions, chmod, chown, reference, octal, rwx]
related:
  - "[[linux/explanation/permissions-model]]"
  - "[[linux/how-to/manage-users]]"
  - "[[linux/reference/cheatsheet]]"
---

# Справочник: Права доступа

## Чтение ls -la

```
-rwxr-xr-- 1 user group 4096 Jan 1 12:00 file.txt
│├──┤├──┤├──┤
│  U    G    O
│  user group other
│
└ тип: - файл, d директория, l ссылка
```

## Octal таблица

| Число | Права | Буквы |
|-------|-------|-------|
| 0 | --- | нет |
| 1 | --x | execute |
| 2 | -w- | write |
| 3 | -wx | write + execute |
| 4 | r-- | read |
| 5 | r-x | read + execute |
| 6 | rw- | read + write |
| 7 | rwx | всё |

## Типичные комбинации

| chmod | Буквы | Назначение |
|-------|-------|-----------|
| `777` | rwxrwxrwx | ⚠️ Все всё могут (не использовать!) |
| `755` | rwxr-xr-x | Директории, скрипты, публичные программы |
| `750` | rwxr-x--- | Директории группы |
| `700` | rwx------ | Приватные директории (~/.ssh/) |
| `644` | rw-r--r-- | Обычные файлы |
| `640` | rw-r----- | Файлы группы |
| `600` | rw------- | Приватные файлы (SSH ключи, .env) |
| `400` | r-------- | Только чтение владельцем |

## Что значат права

| Право | На файле | На директории |
|-------|---------|--------------|
| **r** (read, 4) | Читать содержимое | Листать (`ls`) |
| **w** (write, 2) | Изменять | Создавать/удалять файлы |
| **x** (execute, 1) | Запускать | Входить (`cd`) |

## Специальные биты

| Бит | chmod | ls | Эффект |
|-----|-------|-----|--------|
| SUID | `4755` | `-rwsr-xr-x` | Выполняется от имени владельца файла |
| SGID | `2755` | `drwxr-sr-x` | Файл: от имени группы. Dir: наследование группы |
| Sticky | `1777` | `drwxrwxrwt` | Удалять файлы может только владелец |

## chmod

```bash
chmod 755 file               # числовой
chmod u+x file               # user: +execute
chmod g-w file               # group: -write
chmod o= file                # other: убрать всё
chmod a+r file               # all: +read
chmod -R 755 dir/            # рекурсивно
```

## chown

```bash
chown user file              # сменить владельца
chown user:group file        # владелец + группа
chown -R user:group dir/     # рекурсивно
chgrp group file             # только группу
```

## Критичные файлы

| Файл/Директория | Рекомендуемые права |
|-----------------|-------------------|
| `~/.ssh/` | `700` |
| `~/.ssh/id_rsa` | `600` |
| `~/.ssh/authorized_keys` | `600` |
| `/etc/shadow` | `600` (root:root) |
| `/etc/passwd` | `644` |
| `/etc/sudoers` | `440` (редактировать только через `visudo`) |
| `.env` файлы | `600` |
| SSL private keys | `600` |
