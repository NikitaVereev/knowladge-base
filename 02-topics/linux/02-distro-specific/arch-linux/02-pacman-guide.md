# Pacman Guide

## Overview

Pacman — менеджер пакетов Arch Linux. Быстрый, простой и мощный.

**Что вы узнаете:**
- Основные команды pacman
- Поиск и установка пакетов
- Управление зависимостями
- Обновление системы
- Очистка кэша и orphan пакетов
- Конфигурация pacman

## Prerequisites

- Установленный Arch Linux
- Доступ в интернет
- Базовое знание управления пакетами

## Основные команды

### Синхронизация и обновление

```bash
sudo pacman -Sy                # обновить базу пакетов
sudo pacman -Syuu              # обновить систему (агрессивное, избегайте)
sudo pacman -Syu               # обновить систему (правильный способ)

# Флаги:
# S = sync/install
# y = refresh database
# u = upgrade
```

### Поиск пакетов

```bash
pacman -Ss firefox             # поиск в именах и описаниях
pacman -Ss ^firefox$           # точный поиск
pacman -Si firefox             # информация о пакете
pacman -Sl core                # список пакетов в репозитории
```

### Установка пакетов

```bash
sudo pacman -S package         # установить пакет
sudo pacman -S pkg1 pkg2 pkg3  # несколько пакетов
sudo pacman -S base-devel      # установить группу пакетов
pacman -Sg base-devel          # список пакетов в группе
```

### Удаление пакетов

```bash
sudo pacman -R package         # удалить пакет (оставляет зависимости)
sudo pacman -Rs package        # удалить пакет + зависимости
sudo pacman -Rcs package       # удалить пакет + зависимости + конфиги
```

### Работа с локальными пакетами

```bash
pacman -Q                      # список установленных пакетов
pacman -Qe                     # явно установленные (не зависимости)
pacman -Qd                     # только зависимости
pacman -Qu                     # пакеты которые можно обновить
pacman -Ql package             # файлы из пакета
pacman -Qo /path/to/file       # какой пакет содержит файл
pacman -Qm                     # локальные пакеты (из AUR)
```

### Очистка

```bash
sudo pacman -Sc                # удалить кэш старых версий
sudo pacman -Scc               # удалить весь кэш
sudo pacman -Rs $(pacman -Qdtq) # удалить orphaned пакеты
# Qdtq = Q (query) + d (deps) + t (unrequired) + q (quiet)
```

## Группы пакетов

```bash
pacman -Sg                     # список групп
pacman -Sg base-devel          # пакеты в группе
sudo pacman -S base-devel      # установить всю группу
```

**Популярные группы:**
- `base` — базовые пакеты
- `base-devel` — инструменты для разработки
- `xorg` — X11 display server
- `kde-applications` — KDE приложения

## Конфигурация pacman

**Файл:** `/etc/pacman.conf`

```ini
[options]
HoldPkg = pacman glibc
Architecture = auto
SigLevel = Required DatabaseOptional
ParallelDownloads = 5          # количество параллельных загрузок

# Включите репозитории:
[core]
[extra]
[community]
```

## Makepkg — сборка из исходников

```bash
# Если пакета нет в репо, можно собрать из исходников
wget PKGBUILD_URL
tar xzf package.tar.gz
cd package
makepkg -si                    # собрать и установить

# Флаги:
# s = install dependencies
# i = install when done
# r = remove build dependencies after
```

## Troubleshooting

### Pacman зависает

```bash
# Убейте процесс
sudo pkill -9 pacman

# Удалите блокировку
sudo rm /var/lib/pacman/db.lck
```

### Конфликт пакетов

```bash
sudo pacman -Syu --overwrite '*'  # перезаписать конфликтующие файлы
```

### Orphaned пакеты

```bash
pacman -Qdtq                   # список orphaned
sudo pacman -Rs $(pacman -Qdtq) # удалить их
```

## Key Takeaways

- **-S** для установки (sync), **-R** для удаления, **-Q** для запроса
- **-y** обновить базу, **-u** обновить пакеты, **-yy** переаписать базу
- **Основная команда:** `sudo pacman -Syu` (обновить систему)
- **Очистка:** `sudo pacman -Sc` + `sudo pacman -Rs $(pacman -Qdtq)`

## Related

- [[./03-aur-guide.md|AUR Guide]] — пакеты из пользовательского репозитория
- [[./04-maintenance.md|Maintenance]] — обслуживание системы
- [[../README.md|Arch Index]] — полный индекс

## See Also

- [Pacman Wiki](https://wiki.archlinux.org/title/Pacman)
- [Pacman Tips and Tricks](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks)
