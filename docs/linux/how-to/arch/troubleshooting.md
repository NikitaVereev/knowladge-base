---
title: "Troubleshooting Arch Linux"
type: how-to
tags: [linux, arch, troubleshooting, boot, grub, pacman, drivers, rescue]
sources:
  original: "_inbox/01-linux/02-distro-specific/arch-linux/05-troubleshooting.md"
  wiki: "https://wiki.archlinux.org/title/General_troubleshooting"
related:
  - "[[linux/how-to/arch/maintenance]]"
  - "[[linux/how-to/arch/pacman-and-aur]]"
  - "[[linux/how-to/arch/install]]"
---

# Troubleshooting Arch Linux

> **TL;DR:** Не загружается → chroot с live USB. Pacman сломан → `pacman -Syu` или `pacman-key`.
> Нет сети → `systemctl restart NetworkManager`. Всегда читайте `journalctl -b -p err`.

## Система не загружается

### GRUB не появляется

```bash
# Загрузиться с live USB, затем:
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
arch-chroot /mnt

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
exit && reboot
```

### Зависает при загрузке

```bash
# Добавить в GRUB параметры ядра для диагностики:
# В меню GRUB нажать 'e', в строку linux добавить:
systemd.unit=multi-user.target    # загрузиться без GUI
nomodeset                         # отключить видеодрайвер (если проблема с GPU)

# После загрузки — смотреть логи:
journalctl -b -p err
systemctl --failed
```

### Black screen после загрузки

```bash
# Переключиться в TTY: Ctrl+Alt+F2
# Проблема обычно в видеодрайверах:

# Intel
sudo pacman -S xf86-video-intel

# AMD
sudo pacman -S xf86-video-amdgpu

# NVIDIA
sudo pacman -S nvidia nvidia-utils
```

## Нет интернета

```bash
# Проверить интерфейсы
ip link

# Ethernet
sudo systemctl restart NetworkManager
sudo dhcpcd

# Wi-Fi
nmcli device wifi list
nmcli device wifi connect "SSID" password "PASS"

# DNS не работает
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

## Проблемы с pacman

### Conflicting files

```bash
# Ошибка: "package: /path/to/file exists in filesystem"
# Если уверены что безопасно:
sudo pacman -S --overwrite '*' package

# Или найти владельца:
pacman -Qo /path/to/file
```

### Повреждённая БД или ключи

```bash
# Обновить ключи
sudo pacman-key --init
sudo pacman-key --populate archlinux
sudo pacman-key --refresh-keys

# Пересоздать БД
sudo rm -r /var/lib/pacman/sync
sudo pacman -Syu
```

### Pacman заблокирован

```bash
# "unable to lock database"
sudo rm /var/lib/pacman/db.lck
# Убедитесь что другой pacman не работает: ps aux | grep pacman
```

## Диск заполнен

```bash
# Что занимает место
du -sh /* 2>/dev/null | sort -rh | head -10

# Очистить кэш pacman
sudo pacman -Sc
sudo paccache -rk1

# Очистить журнал
sudo journalctl --vacuum-size=100M

# Найти большие файлы
find / -type f -size +100M 2>/dev/null
```

## Откат обновления

```bash
# Pacman хранит кэш предыдущих версий
ls /var/cache/pacman/pkg/ | grep package-name

# Откатить конкретный пакет
sudo pacman -U /var/cache/pacman/pkg/package-name-oldversion.pkg.tar.zst

# Если система сильно сломана — chroot с live USB
# и откатить проблемные пакеты
```

## Нет звука

```bash
# Проверить PipeWire/PulseAudio
pactl info
systemctl --user status pipewire pipewire-pulse

# Установить если отсутствует
sudo pacman -S pipewire pipewire-pulse wireplumber

# Проверить что не замьючено
wpctl status
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
```

## Общий алгоритм

```
Проблема
  ├─ Система загружается?
  │   ├─ Да → journalctl -b -p err, systemctl --failed
  │   └─ Нет → Live USB → chroot → исправить
  │
  ├─ Связано с обновлением?
  │   ├─ Да → Проверить archlinux.org/news → откатить пакет
  │   └─ Нет → Проверить логи конкретного сервиса
  │
  └─ Не нашли решение?
      └─ Arch Wiki + Arch Forums + journalctl
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Не читают Arch News перед обновлением | archlinux.org/news — обязательно! |
| `pacman -Sy package` (partial upgrade) | Только `pacman -Syu` |
| Удалили важный пакет | `pacman -U /var/cache/pacman/pkg/...` |
| Kernel panic | Загрузить предыдущее ядро из GRUB → Advanced options |
