---
title: "Справочник: Файловая иерархия"
type: reference
tags: [linux, filesystem, fhs, reference, paths, commands]
sources:
  original: "_inbox/01-linux/01-core-concepts/01-filesystem-hierarchy.md"
related:
  - "[[linux/explanation/filesystem]]"
  - "[[linux/reference/cheatsheet]]"
---

# Справочник: Файловая иерархия (FHS)

## Директории

| Путь | Назначение | Пример содержимого |
|------|-----------|-------------------|
| `/` | Корень | Всё начинается здесь |
| `/bin` | Основные команды | ls, cp, grep, cat, mkdir |
| `/sbin` | Системные команды (root) | iptables, fdisk, mkfs |
| `/boot` | Загрузчик и ядро | vmlinuz, initramfs, grub/ |
| `/dev` | Файлы устройств | sda (диск), null, random, tty |
| `/etc` | Конфигурация системы | passwd, hosts, fstab, ssh/ |
| `/home` | Домашние директории | /home/user/ |
| `/root` | Домашняя root | /root/.bashrc |
| `/lib`, `/lib64` | Системные библиотеки | libc.so, modules/ |
| `/usr` | Программы и утилиты | usr/bin/, usr/local/, usr/share/ |
| `/usr/local` | Локально установленное ПО | Компилированное вручную |
| `/var` | Переменные данные | var/log/, var/cache/, var/mail/ |
| `/var/log` | Логи | syslog, auth.log, journal/ |
| `/tmp` | Временные файлы | Очищается при reboot |
| `/proc` | Информация о процессах | /proc/cpuinfo, /proc/1234/ |
| `/sys` | Информация о системе | Драйверы, устройства |
| `/run` | Runtime данные | PID-файлы, сокеты |
| `/opt` | Стороннее ПО | /opt/google/chrome/ |
| `/srv` | Данные сервисов | /srv/www/, /srv/ftp/ |
| `/media` | Автомонтирование | USB, CD |
| `/mnt` | Ручное монтирование | Временные маунты |

## Важные файлы в /etc/

| Файл | Назначение |
|------|-----------|
| `/etc/passwd` | Пользователи (`user:x:uid:gid:name:home:shell`) |
| `/etc/shadow` | Хеши паролей (только root) |
| `/etc/group` | Группы (`group:x:gid:members`) |
| `/etc/sudoers` | Настройки sudo (редактировать через `visudo`) |
| `/etc/hostname` | Имя хоста |
| `/etc/hosts` | Локальный DNS |
| `/etc/fstab` | Монтирование дисков при загрузке |
| `/etc/ssh/sshd_config` | Конфигурация SSH-сервера |
| `/etc/systemd/system/` | Пользовательские systemd-юниты |
| `/etc/apt/sources.list` | Репозитории apt (Debian/Ubuntu) |
| `/etc/pacman.conf` | Конфигурация pacman (Arch) |

## Команды навигации

```bash
pwd                        # текущая директория
ls -la                     # список файлов (включая скрытые)
ls -lh                     # с размерами в human-readable
cd /path                   # перейти
cd ~ / cd                  # домашняя
cd -                       # предыдущая
cd ..                      # на уровень выше
```

## Команды работы с файлами

```bash
# Просмотр
cat file                   # весь файл
head -20 file              # первые 20 строк
tail -20 file              # последние 20 строк
tail -f file               # следить в реальном времени (логи)
less file                  # постраничный просмотр

# Создание
touch file                 # создать пустой файл / обновить время
mkdir -p path/to/dir       # создать директорию (с родительскими)

# Копирование / Перемещение / Удаление
cp source dest             # копировать файл
cp -r dir/ dest/           # копировать директорию
mv source dest             # переместить / переименовать
rm file                    # удалить файл
rm -rf dir/                # удалить директорию рекурсивно

# Поиск
find / -name "*.conf"      # найти по имени
find /var -size +100M      # найти большие файлы
locate filename            # быстрый поиск (по индексу)
which command              # где находится команда
```

## Информация о дисках

```bash
df -h                      # свободное место по разделам
du -sh /path               # размер директории
du -sh /* 2>/dev/null       # что занимает корень
lsblk                      # все блочные устройства (диски)
mount | column -t          # что куда примонтировано
```

## Специальные пути

| Обозначение | Значение |
|-------------|---------|
| `.` | Текущая директория |
| `..` | Родительская директория |
| `~` | Домашняя директория (`/home/user`) |
| `-` | Предыдущая директория (для `cd`) |
| `/dev/null` | «Чёрная дыра» — поглощает всё |
| `/dev/zero` | Бесконечные нули |
| `/dev/random` | Случайные данные |
