# PPA Guide

## Overview

PPA (Personal Package Archive) — способ добавить дополнительные репозитории в Ubuntu.

## Добавление PPA

```bash
sudo add-apt-repository ppa:user/ppa-name
sudo apt update
sudo apt install package
```

## Удаление PPA

```bash
# Способ 1: Через команду
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update

# Способ 2: Вручную
sudo nano /etc/apt/sources.list.d/user-ppa-name-ubuntu-*focal.list  # удалить строку
sudo apt update
```

## Популярные PPA

```bash
# Git latest
sudo add-apt-repository ppa:git-core/ppa

# GIMP latest
sudo add-apt-repository ppa:nilarimogard/webupd8

# VLC latest
sudo add-apt-repository ppa:videolan/master-daily
```

## Предупреждения

⚠️ **Внимание:**
- Не все PPA безопасны
- PPA может содержать unstable версии
- Не смешивайте слишком много PPA
- Проверяйте источник PPA перед добавлением

## Key Takeaways

- **PPA** = Personal Package Archive (пользовательские пакеты)
- `sudo add-apt-repository ppa:user/name` — добавить
- `sudo apt update` — обновить после добавления PPA
- Проверяйте источник перед использованием

## Related

- [[02-apt-guide|APT Guide]]
- [[docs/linux/02-distro-specific/README|Ubuntu Index]]

## See Also

- [Launchpad PPA](https://launchpad.net/ubuntu/+ppas)
- [Ubuntu PPA Wiki](https://help.ubuntu.com/community/Repositories/Ubuntu)
