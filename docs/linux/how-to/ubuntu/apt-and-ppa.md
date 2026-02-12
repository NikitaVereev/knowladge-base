---
title: "apt и PPA"
type: how-to
tags: [linux, ubuntu, apt, ppa, snap, packages, repositories]
sources:
  original: "_inbox/01-linux/02-distro-specific/ubuntu-linux/02-apt-guide.md + 03-ppa-guide.md"
related:
  - "[[linux/how-to/ubuntu/install]]"
  - "[[linux/how-to/ubuntu/maintenance]]"
  - "[[linux/tutorials/02-package-management]]"
---

# apt и PPA

> **TL;DR:** `sudo apt update && sudo apt upgrade` — главная команда.
> PPA — сторонние репозитории. Snap — контейнерные пакеты (альтернатива apt).

## apt — основные команды

```bash
# Обновить индекс + обновить все пакеты
sudo apt update && sudo apt upgrade

# Полное обновление (может удалять пакеты для разрешения зависимостей)
sudo apt full-upgrade

# Установить
sudo apt install nginx git vim

# Удалить
sudo apt remove nginx              # оставить конфиги
sudo apt purge nginx               # удалить с конфигами

# Очистка
sudo apt autoremove                # неиспользуемые зависимости
sudo apt autoclean                 # старые .deb из кэша
sudo apt clean                     # весь кэш

# Поиск и информация
apt search nginx
apt show nginx
apt list --installed
apt list --upgradable
```

## Репозитории

Ubuntu имеет 4 категории:
- **main** — поддерживается Canonical, свободное ПО
- **universe** — поддерживается сообществом
- **restricted** — проприетарные драйверы
- **multiverse** — несвободное ПО

```bash
# Файл репозиториев
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
```

## PPA (Personal Package Archive)

Сторонние репозитории на Launchpad. Содержат свежие версии или ПО отсутствующее в main.

```bash
# Добавить PPA
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git

# Удалить PPA
sudo add-apt-repository --remove ppa:git-core/ppa
sudo apt update
```

> **⚠️ Безопасность:** Проверяйте источник PPA. Не добавляйте незнакомые PPA — они могут содержать вредоносный код или нестабильные версии.

## Snap (альтернатива apt)

```bash
# Snap предустановлен в Ubuntu
snap install package               # установить
snap list                          # установленные
snap refresh                       # обновить все
snap remove package                # удалить

# Примеры
snap install code --classic        # VS Code
snap install firefox
snap install telegram-desktop
```

Snap vs apt: snap — контейнеризованные пакеты (изолированные, с зависимостями внутри), занимают больше места, обновляются автоматически.

## Ubuntu-специфичные команды

```bash
# Установить проприетарные драйверы
sudo ubuntu-drivers autoinstall

# Выбрать программу по умолчанию
sudo update-alternatives --config editor
sudo update-alternatives --config java

# Информация о версии
lsb_release -a
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `Unable to locate package` | `sudo apt update` перед install |
| `Could not get lock /var/lib/dpkg/lock` | Другой apt работает. `sudo kill PID` или подождать |
| `dpkg was interrupted` | `sudo dpkg --configure -a` |
| Broken dependencies | `sudo apt -f install` |
| PPA для другой версии Ubuntu | Удалить PPA, найти подходящую версию |
