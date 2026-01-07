# APT Guide for Ubuntu

## Overview

APT (Advanced Package Tool) — менеджер пакетов Ubuntu. Основной способ установки ПО.

## Basic Commands

```bash
sudo apt update              # обновить список пакетов
sudo apt upgrade             # обновить установленные пакеты
sudo apt full-upgrade        # более агрессивное обновление
sudo apt install package     # установить пакет
sudo apt remove package      # удалить пакет
sudo apt purge package       # удалить с конфигами
sudo apt autoremove          # удалить неиспользуемые зависимости
sudo apt search term         # поиск пакета
apt show package             # информация о пакете
apt list --installed         # список установленных
apt list --upgradable        # что можно обновить
```

## Ubuntu специфичные команды

```bash
sudo ubuntu-drivers autoinstall    # установить proprietary драйверы
sudo update-alternatives           # выбрать программу по умолчанию
snap install package               # установить snap (альтернатива apt)
snap list                          # список snap пакетов
```

## Работа с репозиториями

```bash
add-apt-repository ppa:user/ppa-name    # добавить PPA
sudo add-apt-repository --remove ppa:user/ppa-name  # удалить PPA
apt-get source package                  # скачать исходники
```

## Очистка

```bash
sudo apt clean               # удалить кэш .deb файлов
sudo apt autoclean           # удалить старые версии
sudo apt autoremove          # удалить неиспользуемые пакеты
sudo apt autoremove --purge  # + удалить конфиги
```

## Key Takeaways

- `sudo apt update && sudo apt upgrade` — главная команда
- Ubuntu + apt просто и понятно
- apt работает с репозиториями (main, universe, multiverse)

## Related

- [[./03-ppa-guide.md|PPA Guide]]
- [[../README.md|Ubuntu Index]]

## See Also

- [APT Manual](https://linux.die.net/man/8/apt)
- [Ubuntu Package Management](https://help.ubuntu.com/community/AptGetHowto)
