---
created: 2026-01-05
updated: 2026-01-05
type: reference
---

# Управление пакетами

## Основы

**Пакет** — архив с программой, библиотекой или модулем.

**Пакетный менеджер** — инструмент для установки, обновления, удаления пакетов.

**Репозиторий** — сервер, хранящий пакеты.

**Зависимости** — другие пакеты, требуемые для работы.

---

## apt (Debian/Ubuntu)

### Базовые команды

```bash
sudo apt update              # обновить список пакетов
sudo apt upgrade             # обновить установленные пакеты
sudo apt full-upgrade        # более агрессивное обновление
sudo apt install package     # установить пакет
sudo apt remove package      # удалить пакет (оставляет конфиги)
sudo apt purge package       # удалить пакет полностью
sudo apt autoremove          # удалить неиспользуемые зависимости
sudo apt search package      # поиск пакета
sudo apt show package        # информация о пакете
apt list --installed         # список установленных пакетов
apt list --upgradable        # пакеты с обновлениями
```

### Примеры

```bash
sudo apt update && sudo apt upgrade -y  # обновить всё
sudo apt install curl wget git          # установить несколько
sudo apt remove firefox                 # удалить firefox
sudo apt autoremove                     # очистить мусор
sudo apt search python3                 # найти python3 пакеты
```

### Проблемы

```bash
# Ошибка: "E: Could not get lock"
sudo rm /var/lib/apt/lists/lock
sudo apt update

# Ошибка: "Some packages could not be installed"
sudo apt install -f                     # исправить зависимости
sudo apt --fix-broken install

# Очистить кэш
sudo apt clean
sudo apt autoclean
```

---

## dnf (Fedora/RHEL)

### Базовые команды

```bash
sudo dnf update              # обновить список и установленные пакеты
sudo dnf install package     # установить пакет
sudo dnf remove package      # удалить пакет
sudo dnf search package      # поиск пакета
sudo dnf info package        # информация о пакете
sudo dnf list installed      # список установленных
sudo dnf list upgrades       # пакеты с обновлениями
sudo dnf autoremove          # удалить неиспользуемые
```

### Примеры

```bash
sudo dnf update -y                      # обновить всё
sudo dnf install curl wget git          # установить несколько
sudo dnf remove firefox                 # удалить
sudo dnf groupinstall "Development Tools"  # группа пакетов
```

### Репозитории

```bash
sudo dnf repolist            # список репозиториев
sudo dnf repolist all        # включая отключённые
sudo dnf config-manager --add-repo URL  # добавить репозиторий
```

---

## pacman (Arch Linux)

### Базовые команды

```bash
sudo pacman -Syu             # обновить всё (S=sync, y=refresh, u=upgrade)
sudo pacman -S package       # установить пакет
sudo pacman -R package       # удалить пакет
sudo pacman -Rs package      # удалить + зависимости
sudo pacman -Ss package      # поиск пакета
sudo pacman -Si package      # информация о пакете
pacman -Q                    # список установленных
pacman -Qu                   # пакеты с обновлениями
sudo pacman -Sc              # очистить кэш
```

### Примеры

```bash
sudo pacman -Syu                        # обновить систему
sudo pacman -S firefox chromium         # установить браузеры
sudo pacman -R firefox                  # удалить firefox
sudo pacman -Ss "python"                # найти python пакеты
```

### Очистка

```bash
sudo pacman -Sc              # удалить неиспользуемые версии
sudo pacman -Scc             # очистить весь кэш (осторожно!)
sudo pacman -Rs $(pacman -Qdtq)  # удалить все orphan'ы
```

---

## Таблица сравнения

| Задача | apt | dnf | pacman |
|--------|-----|-----|--------|
| Обновить | `apt upgrade` | `dnf upgrade` | `pacman -Syu` |
| Установить | `apt install` | `dnf install` | `pacman -S` |
| Удалить | `apt remove` | `dnf remove` | `pacman -R` |
| Поиск | `apt search` | `dnf search` | `pacman -Ss` |
| Информация | `apt show` | `dnf info` | `pacman -Si` |
| Очистка | `apt autoremove` | `dnf autoremove` | `pacman -Rs` |

---

## Поиск пакетов онлайн

```
Debian/Ubuntu: https://packages.debian.org, https://packages.ubuntu.com
Fedora: https://packages.fedoraproject.org
Arch: https://archlinux.org/packages/
```

---

## Установка из исходников

Если пакета нет в репозитории:

```bash
# Загрузить
wget https://example.com/package.tar.gz
tar -xzf package.tar.gz
cd package

# Собрать и установить
./configure          # настройка
make                 # компиляция
sudo make install    # установка
```

**Требования:**
```bash
# Ubuntu/Debian
sudo apt install build-essential

# Fedora
sudo dnf groupinstall "Development Tools"

# Arch
sudo pacman -S base-devel
```

---

## pip (Python пакеты)

```bash
pip install package         # установить
pip install package==1.0    # конкретную версию
pip install -r requirements.txt  # из файла
pip list                    # список установленных
pip search package          # поиск
pip uninstall package       # удалить
```

**Виртуальное окружение:**
```bash
python3 -m venv env         # создать окружение
source env/bin/activate     # активировать
pip install package         # устанавливать в окружение
deactivate                  # выключить
```

---

## npm (Node.js пакеты)

```bash
npm install package         # установить локально
npm install -g package      # установить глобально
npm list                    # список установленных
npm search package          # поиск
npm uninstall package       # удалить
npm update                  # обновить
```

---

## Проблемы и решения

### Не могу найти пакет

```bash
# 1. Обновить список репозиториев
sudo apt update             # apt
sudo dnf update             # dnf

# 2. Поискать с другим названием
apt search python           # может быть python3
```

### Конфликт версий

```bash
# apt
sudo apt install package=version

# dnf
sudo dnf install package-version

# pacman
sudo pacman -S package=version
```

### Сломанные зависимости

```bash
# apt
sudo apt --fix-broken install
sudo apt install -f

# dnf
sudo dnf install -y --skip-broken

# pacman
sudo pacman -Syyu
```

### Откатить обновление

```bash
# apt: посмотреть историю
sudo apt full-upgrade --simulate
sudo apt install package=old_version

# dnf: посмотреть историю
sudo dnf history
sudo dnf downgrade package

# pacman: посмотреть в архиве
sudo pacman -U /var/cache/pacman/pkg/package-old_version.pkg.tar.zst
```

### Очистить место

```bash
# Удалить кэш
sudo apt clean               # apt
sudo dnf clean all           # dnf
sudo pacman -Sc              # pacman

# Удалить старые версии
sudo apt autoremove          # apt
sudo dnf autoremove          # dnf
sudo pacman -Rs $(pacman -Qdtq)  # pacman
```

---

## Шпаргалка

```bash
# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установить программу
sudo apt install program
sudo dnf install program
sudo pacman -S program

# Поиск пакета
apt search term
dnf search term
pacman -Ss term

# Удалить пакет
sudo apt remove program
sudo dnf remove program
sudo pacman -R program

# Очистить мусор
sudo apt autoremove
sudo dnf autoremove
sudo pacman -Rs $(pacman -Qdtq)
```

---

## Дальше

[Специфика дистрибутивов](../02-distro-specific/README.md) или [Администрирование](../03-system-administration/README.md)
