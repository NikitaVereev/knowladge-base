---
title: "Troubleshooting Ubuntu"
type: how-to
tags: [linux, ubuntu, troubleshooting, boot, grub, apt, snap, drivers]
sources:
  original: "_inbox/01-linux/02-distro-specific/ubuntu-linux/05-troubleshooting.md"
related:
  - "[[linux/how-to/ubuntu/maintenance]]"
  - "[[linux/how-to/ubuntu/apt-and-ppa]]"
---

# Troubleshooting Ubuntu

> **TL;DR:** Не загружается → Recovery Mode через GRUB. apt сломан → `sudo dpkg --configure -a`.
> Нет сети → `sudo systemctl restart NetworkManager`. Логи: `journalctl -b -p err`.

## Система не загружается

```bash
# Recovery Mode:
# При загрузке в GRUB → Advanced options → Recovery mode
# → root (Drop to root shell prompt)

# Remount filesystem с записью
mount -o remount,rw /

# Исправить пакеты
apt update && apt upgrade
dpkg --configure -a

# Переустановить GRUB
grub-install /dev/sda
update-grub
reboot
```

## Нет интернета

```bash
# Проверить
ip a
ping -c 3 8.8.8.8

# Перезапустить NetworkManager
sudo systemctl restart NetworkManager

# Wi-Fi
nmcli device wifi list
nmcli device wifi connect "SSID" password "PASS"

# DNS
sudo systemctl restart systemd-resolved
resolvectl status
```

## Проблемы с apt

### Broken dependencies

```bash
sudo apt -f install                # исправить зависимости
sudo dpkg --configure -a           # доконфигурировать пакеты
sudo apt update --fix-missing
```

### Lock file

```bash
# "Could not get lock"
# 1. Подождать завершения другого процесса apt
# 2. Если точно ничего не работает:
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/apt/lists/lock
sudo dpkg --configure -a
```

## Проблемы со Snap

```bash
# Snap не запускается
snap refresh
snap disable package && snap enable package

# Snap занимает много места
snap list --all
# Удалить старые ревизии:
snap remove package --revision=N
```

## Проблемы с драйверами / графикой

```bash
# Установить рекомендуемые драйверы
sudo ubuntu-drivers autoinstall

# Или конкретный
sudo ubuntu-drivers list
sudo apt install nvidia-driver-535

# Если GUI не загружается — переключиться в TTY:
# Ctrl+Alt+F2
sudo apt install --reinstall ubuntu-desktop
```

## Диск заполнен

```bash
# Найти что занимает
du -sh /* 2>/dev/null | sort -rh | head
du -sh /var/log/*
du -sh /snap/*

# Очистить
sudo apt clean
sudo apt autoremove --purge
sudo journalctl --vacuum-size=100M
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| «Ubuntu won't boot» | GRUB → Recovery mode → root → `apt upgrade` |
| `dpkg was interrupted` | `sudo dpkg --configure -a` |
| PPA сломал систему | `sudo ppa-purge ppa:user/name` (нужен `ppa-purge`) |
| Обновление прервано | `sudo apt -f install && sudo dpkg --configure -a` |
| Чёрный экран после обновления | TTY (Ctrl+Alt+F2) → переустановить драйверы |