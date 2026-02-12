---
title: "Файловая система Linux"
type: explanation
tags: [linux, filesystem, fhs, hierarchy, paths, symlinks, types]
sources:
  original: "_inbox/01-linux/01-core-concepts/01-filesystem-hierarchy.md"
related:
  - "[[linux/reference/filesystem-hierarchy]]"
  - "[[linux/tutorials/01-getting-started]]"
  - "[[linux/explanation/permissions-model]]"
---

# Файловая система Linux

> **TL;DR:** Всё — файл. Единый корень `/`. FHS стандартизирует структуру на всех дистрибутивах.
> `/etc` — конфиги, `/home` — пользователи, `/var/log` — логи, `/tmp` — временное.

## Философия: всё — файл

В Linux всё представлено как файл: обычные файлы, директории, устройства (`/dev/sda`), процессы (`/proc/1234`), сетевые сокеты. Это позволяет работать с любым ресурсом одними инструментами (`cat`, `echo`, `ls`).

## FHS (Filesystem Hierarchy Standard)

Все дистрибутивы следуют единому стандарту:

```
/                          корень (root directory)
├── /bin/                  основные команды (ls, cp, grep)
├── /sbin/                 системные команды (ifconfig, iptables)
├── /boot/                 kernel, bootloader (GRUB)
├── /dev/                  файлы устройств (sda, tty, null)
├── /etc/                  конфигурация системы
│   ├── passwd             пользователи (user:x:uid:gid:name:home:shell)
│   ├── shadow             хеши паролей (только root)
│   ├── group              группы
│   ├── hosts              локальный DNS
│   ├── fstab              монтирование дисков при загрузке
│   ├── sudoers            кто может sudo
│   └── systemd/system/    юниты systemd
├── /home/                 домашние директории пользователей
├── /root/                 домашняя root-пользователя
├── /lib/, /lib64/         системные библиотеки
├── /usr/                  пользовательские программы
│   ├── bin/               команды пользователей
│   ├── local/             локально установленное ПО
│   └── share/             общие данные (документация, иконки)
├── /var/                  переменные данные
│   ├── log/               логи системы и приложений
│   ├── cache/             кэши
│   └── tmp/               временные файлы приложений
├── /tmp/                  временные файлы (очищается при reboot)
├── /proc/                 процессы (виртуальная FS, в памяти)
├── /sys/                  информация о системе (виртуальная FS)
├── /run/                  runtime данные (PID, sockets)
├── /opt/                  стороннее ПО (большие приложения)
├── /srv/                  данные сервисов
├── /media/                автомонтирование (USB, CD)
└── /mnt/                  ручное монтирование
```

## Абсолютные vs относительные пути

**Абсолютный** — от корня: `/home/user/documents/file.txt`. Однозначный, работает из любого места.

**Относительный** — от текущей директории: `../documents/file.txt`. Короче, но зависит от pwd.

Специальные обозначения: `.` — текущая директория, `..` — родительская, `~` — домашняя (`/home/user`), `-` — предыдущая (для `cd -`).

## Типы файлов

| Символ | Тип | Пример |
|--------|-----|--------|
| `-` | Обычный файл | `/etc/hosts` |
| `d` | Директория | `/home/user/` |
| `l` | Символическая ссылка | `/usr/bin/python → python3` |
| `b` | Блочное устройство | `/dev/sda` (диск) |
| `c` | Символьное устройство | `/dev/tty` (терминал) |
| `s` | Сокет | `/run/docker.sock` |
| `p` | Именованный канал (pipe) | FIFO-файлы |

```bash
# Первый символ в ls -l показывает тип
ls -la /dev/sda
# brw-rw---- 1 root disk 8, 0  /dev/sda
# ^-- b = block device
```

## Символические ссылки (symlinks)

```bash
# Создать
ln -s /path/to/target /path/to/link

# Пример: альтернативная версия Python
ln -s /usr/bin/python3.12 /usr/local/bin/python

# Проверить куда ведёт
readlink -f /usr/local/bin/python
ls -la /usr/local/bin/python
# lrwxrwxrwx ... python -> /usr/bin/python3.12
```

Symlink — указатель на другой файл. Если цель удалена — ссылка «висит» (broken symlink).

## Скрытые файлы

Файлы начинающиеся с `.` — скрытые. Конфигурация хранится в домашней директории:

```
~/.bashrc          конфигурация bash
~/.ssh/            SSH ключи и конфиги
~/.config/         конфигурация приложений (XDG)
~/.local/          данные приложений
~/.gitconfig       конфигурация git
```

```bash
ls -la ~           # показать скрытые файлы
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| «Не нахожу файл» | `find / -name "filename" 2>/dev/null` или `locate filename` |
| «Нет доступа» | Проверить права: `ls -la`, использовать `sudo` для системных файлов |
| «Диск заполнен» | `df -h` (свободное место), `du -sh /var/*` (что занимает) |
| «Удалил важный файл» | В Linux нет корзины. Используйте бэкапы. `extundelete` — крайний случай |
