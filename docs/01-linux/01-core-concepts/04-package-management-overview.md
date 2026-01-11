# Package Management Overview

## Overview

Управление пакетами - это способ установки, обновления и удаления программного обеспечения в Linux. Разные дистрибутивы используют разные пакетные менеджеры.

**Что вы узнаете:**
- Что такое пакеты, менеджеры, репозитории и зависимости
- apt (Debian/Ubuntu) - основные команды
- dnf (Fedora/RHEL) - основные команды
- pacman (Arch Linux) - основные команды
- Установка из исходников (./configure, make)
- pip (Python) и npm (Node.js) пакеты
- Решение проблем с зависимостями

## Prerequisites

Перед этим разделом нужно:
- Понимание файловой системы ([[01-filesystem-hierarchy|Filesystem Hierarchy]])
- Знание пользователей и sudo ([[02-users-groups-permissions|Users and Permissions]])
- Знание сервисов systemd ([[03-processes-and-services|Processes and Services]])

## Package Management Concepts

**Пакет** — архив (обычно .deb, .rpm, или .tar.gz) содержащий программу, библиотеку или модуль.

**Пакетный менеджер** — инструмент для установки, обновления, удаления пакетов (apt, dnf, pacman).

**Репозиторий** — сервер (в интернете), хранящий пакеты (например: archive.ubuntu.com).

**Зависимости** — другие пакеты, требуемые для работы программы (менеджер устанавливает их автоматически).

## apt (Debian/Ubuntu)

**apt** (Advanced Package Tool) — пакетный менеджер для Debian/Ubuntu.

### Basic Commands

```bash
sudo apt update              # обновить список пакетов из репозиториев
sudo apt upgrade             # обновить установленные пакеты
sudo apt full-upgrade        # более агрессивное обновление (может удалить пакеты)
sudo apt install package     # установить пакет
sudo apt remove package      # удалить пакет (оставляет конфиги)
sudo apt purge package       # удалить пакет полностью (с конфигами)
sudo apt autoremove          # удалить неиспользуемые зависимости
sudo apt search package      # поиск пакета
sudo apt show package        # информация о пакете
apt list --installed         # список установленных пакетов
apt list --upgradable        # пакеты которые можно обновить
```

### Examples

```bash
# Обновить систему
sudo apt update
sudo apt upgrade -y

# Установить программы
sudo apt install firefox chromium git vim

# Найти пакет
apt search python3
apt search "web server"

# Информация о пакете
apt show nginx
# показывает: описание, версия, размер, зависимости и т.д.

# Удалить программу
sudo apt remove firefox

# Полная очистка
sudo apt autoremove
sudo apt clean
```

### Troubleshooting

```bash
# Ошибка: "E: Could not get lock"
# Apt уже запущен или нужно перезагрузиться
sudo rm /var/lib/apt/lists/lock
sudo apt update

# Ошибка: "Some packages could not be installed"
# Сломанные зависимости
sudo apt install -f
sudo apt --fix-broken install

# Очистить место
sudo apt clean              # удалить кэш пакетов
sudo apt autoclean          # удалить старые версии
```

## dnf (Fedora/RHEL)

**dnf** (Dandified Yum) — пакетный менеджер для Fedora/RHEL (приемник yum).

### Basic Commands

```bash
sudo dnf update              # обновить список И установленные пакеты
sudo dnf install package     # установить пакет
sudo dnf remove package      # удалить пакет
sudo dnf search package      # поиск пакета
sudo dnf info package        # информация о пакете
sudo dnf list installed      # список установленных
sudo dnf list upgrades       # пакеты с обновлениями
sudo dnf autoremove          # удалить неиспользуемые
sudo dnf groupinstall "name" # установить группу пакетов
```

### Examples

```bash
# Обновить систему
sudo dnf update -y

# Установить программы
sudo dnf install firefox vim git

# Группы пакетов
sudo dnf groupinstall "Development Tools"
sudo dnf groupinstall "GNOME Desktop"

# Удалить
sudo dnf remove firefox

# Информация
dnf info nginx
dnf search python
```

### Repositories

```bash
sudo dnf repolist              # список включённых репозиториев
sudo dnf repolist all          # все репозитории (включая отключённые)
sudo dnf config-manager --add-repo URL  # добавить репозиторий
sudo dnf config-manager --disable repo  # отключить репозиторий
```

## pacman (Arch Linux)

**pacman** — пакетный менеджер для Arch Linux (очень быстрый и простой).

### Basic Commands

```bash
sudo pacman -Syu             # обновить всё (S=sync, y=refresh, u=upgrade)
sudo pacman -S package       # установить пакет
sudo pacman -R package       # удалить пакет (оставляет зависимости)
sudo pacman -Rs package      # удалить пакет + зависимости
sudo pacman -Ss package      # поиск пакета
sudo pacman -Si package      # информация о пакете
pacman -Q                    # список установленных
pacman -Qu                   # пакеты с обновлениями
sudo pacman -Sc              # очистить кэш старых версий
```

### Examples

```bash
# Обновить систему
sudo pacman -Syu

# Установить программы
sudo pacman -S firefox vim git

# Удалить с зависимостями
sudo pacman -Rs firefox

# Полная очистка
sudo pacman -Sc              # удалить неиспользуемые версии
sudo pacman -Scc             # очистить весь кэш (осторожно!)
sudo pacman -Rs $(pacman -Qdtq)  # удалить orphaned пакеты
```

## Comparison Table

| Задача | apt (Debian) | dnf (Fedora) | pacman (Arch) |
|--------|---|---|---|
| Обновить систему | `apt update && apt upgrade` | `dnf update` | `pacman -Syu` |
| Установить | `apt install PKG` | `dnf install PKG` | `pacman -S PKG` |
| Удалить | `apt remove PKG` | `dnf remove PKG` | `pacman -R PKG` |
| Поиск | `apt search TERM` | `dnf search TERM` | `pacman -Ss TERM` |
| Информация | `apt show PKG` | `dnf info PKG` | `pacman -Si PKG` |
| Список установленных | `apt list --installed` | `dnf list installed` | `pacman -Q` |
| Очистка | `apt autoremove` | `dnf autoremove` | `pacman -Rs orphans` |

## Installing from Source

Если пакета нет в репозитории, можно установить из исходного кода:

```bash
# 1. Скачать архив
wget https://example.com/package.tar.gz
tar -xzf package.tar.gz
cd package

# 2. Собрать
./configure              # настройка (проверяет зависимости)
make                     # компиляция
sudo make install        # установка

# 3. Очистить (опционально)
make clean               # удалить объектные файлы
cd .. && rm -rf package  # удалить исходный код
```

### Build Tools

Требуется пакет с инструментами для компиляции:

```bash
# Ubuntu/Debian
sudo apt install build-essential

# Fedora/RHEL
sudo dnf groupinstall "Development Tools"

# Arch
sudo pacman -S base-devel
```

## pip - Python Package Manager

```bash
pip install package              # установить пакет
pip install package==1.2.3       # конкретную версию
pip install -r requirements.txt  # из файла
pip list                         # список установленных
pip show package                 # информация о пакете
pip uninstall package            # удалить пакет
pip search package               # поиск (может не работать)
pip freeze > requirements.txt    # сохранить список в файл
```

### Virtual Environments

**Виртуальное окружение** — изолированная среда Python для проекта:

```bash
python3 -m venv env              # создать окружение (env папка)
source env/bin/activate          # активировать (на Linux/Mac)
env\Scripts\activate             # активировать (на Windows)
pip install -r requirements.txt  # устанавливать в окружение
deactivate                       # выключить окружение
```

**Зачем нужны:**
- Разные проекты могут требовать разные версии одного пакета
- Не загрязняем систему Python
- Легче поделиться проектом (requirements.txt)

## npm - Node.js Package Manager

```bash
npm install package              # установить локально (в папку node_modules)
npm install -g package           # установить глобально (для всей системы)
npm install                      # установить из package.json
npm list                         # список установленных
npm search package               # поиск пакета
npm update package               # обновить пакет
npm uninstall package            # удалить пакет
npm init                         # создать package.json для проекта
```

## Finding Packages Online

Если не знаете точное название пакета:

**Debian/Ubuntu:**
- https://packages.debian.org
- https://packages.ubuntu.com

**Fedora/RHEL:**
- https://packages.fedoraproject.org

**Arch:**
- https://archlinux.org/packages/

**Python:**
- https://pypi.org

**Node.js:**
- https://www.npmjs.com

## Troubleshooting

### Не могу найти пакет

```bash
# 1. Обновите список пакетов
sudo apt update                  # apt
sudo dnf update                  # dnf

# 2. Поищите с другим названием
apt search "web server"          # apt
dnf search python                # dnf

# 3. Проверьте что репозитории включены
sudo dnf repolist                # dnf
```

### Конфликт версий

```bash
# apt: установить конкретную версию
sudo apt install package=1.2.3

# dnf: установить конкретную версию
sudo dnf install package-1.2.3

# pacman: обычно версия одна
sudo pacman -S package
```

### Сломанные зависимости

```bash
# apt
sudo apt install -f
sudo apt --fix-broken install

# dnf
sudo dnf install -y --skip-broken
sudo dnf install -x package --skip-broken

# pacman
sudo pacman -Syyu
```

### Нужна версия которая была раньше

```bash
# apt: проверить историю
sudo apt full-upgrade --simulate
sudo apt install package=old_version

# dnf: посмотреть историю
sudo dnf history
sudo dnf downgrade package

# pacman: посмотреть кэш
ls /var/cache/pacman/pkg/package-*
sudo pacman -U /var/cache/pacman/pkg/package-old_version.pkg.tar.zst
```

### Много места занято

```bash
# Очистить кэш пакетов
sudo apt clean               # apt
sudo dnf clean all           # dnf
sudo pacman -Sc              # pacman

# Удалить неиспользуемые пакеты
sudo apt autoremove          # apt
sudo dnf autoremove          # dnf
sudo pacman -Rs $(pacman -Qdtq)  # pacman
```

## Cheat Sheet

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y
sudo dnf update -y
sudo pacman -Syu

# Установить программу
sudo apt install program
sudo dnf install program
sudo pacman -S program

# Поиск пакета
apt search term
dnf search term
pacman -Ss term

# Информация о пакете
apt show package
dnf info package
pacman -Si package

# Удалить пакет
sudo apt remove program
sudo dnf remove program
sudo pacman -R program

# Очистить место
sudo apt autoremove && sudo apt clean
sudo dnf autoremove && sudo dnf clean all
sudo pacman -Rs $(pacman -Qdtq) && sudo pacman -Sc

# Python
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
deactivate

# Node.js
npm install                  # из package.json
npm install package          # конкретный пакет
npm list                     # что установлено
```

## Key Takeaways

- **apt** — для Debian/Ubuntu (самый популярный)
- **dnf** — для Fedora/RHEL (enterprise)
- **pacman** — для Arch Linux (простой и быстрый)
- **apt update** — обновить список пакетов (НУЖНО ДО УСТАНОВКИ!)
- **зависимости** — менеджер устанавливает автоматически
- **pip/npm** — для языков программирования (Python, Node.js)
- **из исходников** — ./configure, make, make install (если нет в репо)

## Related

Предыдущий шаг:
- [[03-processes-and-services|Processes and Services]] — управление сервисами

Следующие шаги:
- [[docs/01-linux/02-distro-specific/README|Distro-Specific]] — углубленные инструкции для дистрибутива
- [[docs/01-linux/03-system-administration/README|System Administration]] — администрирование

Контекст:
- [[docs/01-linux/01-core-concepts/README|Core Concepts Index]] — полный индекс этого раздела

## See Also

Встроенная справка:
- `apt help` — справка по apt
- `man apt` — полная справка по apt
- `dnf help` — справка по dnf
- `man pacman` — справка по pacman

Онлайн ресурсы:
- [Debian Package Management](https://wiki.debian.org/PackageManagement) — Debian Wiki
- [Fedora Package Management](https://docs.fedoraproject.org/en-US/quick-docs/installing-software/) — Fedora docs
- [Arch Pacman](https://wiki.archlinux.org/title/Pacman) — Arch Wiki
- [PyPI - Python Package Index](https://pypi.org) — репо Python пакетов
- [npm Registry](https://www.npmjs.com) — репо Node.js пакетов
