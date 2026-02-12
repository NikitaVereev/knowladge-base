---
title: "02 — Управление пакетами"
type: tutorial
tags: [linux, tutorial, packages, apt, pacman, dnf, pip, npm, install]
sources:
  original: "_inbox/01-linux/01-core-concepts/04-package-management-overview.md"
related:
  - "[[linux/tutorials/01-getting-started]]"
  - "[[linux/how-to/arch/pacman-and-aur]]"
  - "[[linux/how-to/ubuntu/apt-and-ppa]]"
  - "[[linux/how-to/manage-packages]]"
---

# Tutorial 02 — Управление пакетами

> **Цель:** Научиться искать, устанавливать, обновлять и удалять пакеты.
> Три пакетных менеджера: apt (Debian/Ubuntu), pacman (Arch), dnf (Fedora/RHEL).

**Время:** ~25 минут
**Требования:** Пройден Tutorial 01. Терминал.

## Шаг 1. Концепция

Пакет — архив с программой, её зависимостями и метаданными. Пакетный менеджер скачивает из репозитория (удалённого хранилища), разрешает зависимости и устанавливает.

```
Вы: "Установи nginx"
       ↓
Пакетный менеджер:
  1. Ищет nginx в репозитории
  2. Находит зависимости (libssl, libpcre)
  3. Скачивает nginx + зависимости
  4. Устанавливает всё
  5. Настраивает
```

## Шаг 2. apt (Debian, Ubuntu, Mint)

```bash
# Обновить список пакетов (делать перед install!)
sudo apt update

# Обновить все установленные пакеты
sudo apt upgrade

# Найти пакет
apt search nginx

# Информация о пакете
apt show nginx

# Установить
sudo apt install nginx

# Установить несколько
sudo apt install git vim curl wget

# Удалить (оставить конфиги)
sudo apt remove nginx

# Удалить полностью (с конфигами)
sudo apt purge nginx

# Удалить неиспользуемые зависимости
sudo apt autoremove

# Список установленных
apt list --installed | grep nginx
```

## Шаг 3. pacman (Arch, Manjaro)

```bash
# Синхронизировать + обновить всё
sudo pacman -Syu

# Найти пакет
pacman -Ss nginx

# Информация
pacman -Si nginx             # из репозитория
pacman -Qi nginx             # установленный

# Установить
sudo pacman -S nginx

# Удалить (с зависимостями)
sudo pacman -Rs nginx

# Список установленных
pacman -Q | grep nginx

# Какому пакету принадлежит файл?
pacman -Qo /usr/bin/nginx

# Очистить кэш
sudo pacman -Sc
```

## Шаг 4. dnf (Fedora, RHEL, Rocky)

```bash
# Обновить все пакеты
sudo dnf upgrade

# Найти
dnf search nginx

# Информация
dnf info nginx

# Установить
sudo dnf install nginx

# Удалить
sudo dnf remove nginx

# Группы пакетов
dnf group list
sudo dnf group install "Development Tools"

# Список установленных
dnf list installed | grep nginx
```

## Шаг 5. Практика — установите полезные утилиты

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install htop tree jq curl wget git vim tmux

# Arch
sudo pacman -S htop tree jq curl wget git vim tmux

# Fedora
sudo dnf install htop tree jq curl wget git vim tmux
```

Проверьте:
```bash
htop              # интерактивный мониторинг (q для выхода)
tree -L 2 /etc    # дерево директорий
echo '{"a":1}' | jq .   # форматирование JSON
```

## Шаг 6. pip и npm (для разработчиков)

```bash
# Python
pip install requests           # установить пакет
pip install -r requirements.txt # из файла
pip list                       # установленные
python -m venv .venv           # виртуальное окружение (РЕКОМЕНДУЕТСЯ)
source .venv/bin/activate
pip install flask

# Node.js
npm install express            # в текущий проект
npm install -g nodemon         # глобально
npm list
```

## Сводная таблица

| Действие | apt (Debian) | pacman (Arch) | dnf (Fedora) |
|----------|-------------|---------------|-------------|
| Обновить индекс | `apt update` | `pacman -Sy` | `dnf check-update` |
| Обновить всё | `apt upgrade` | `pacman -Syu` | `dnf upgrade` |
| Установить | `apt install pkg` | `pacman -S pkg` | `dnf install pkg` |
| Удалить | `apt remove pkg` | `pacman -Rs pkg` | `dnf remove pkg` |
| Найти | `apt search pkg` | `pacman -Ss pkg` | `dnf search pkg` |
| Информация | `apt show pkg` | `pacman -Si pkg` | `dnf info pkg` |
| Установленные | `apt list --installed` | `pacman -Q` | `dnf list installed` |
| Очистка | `apt autoremove` | `pacman -Sc` | `dnf autoremove` |

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `Unable to locate package` | `sudo apt update` перед install |
| `Could not get lock` | Другой процесс apt работает. Подождать или `sudo kill PID` |
| Конфликт зависимостей | `sudo apt -f install` (Debian) или `pacman -Syu` (Arch) |
| `pip install` без venv загрязняет систему | Всегда `python -m venv .venv` |

## Что дальше

→ Подробнее: [[linux/how-to/arch/pacman-and-aur]] или [[linux/how-to/ubuntu/apt-and-ppa]]