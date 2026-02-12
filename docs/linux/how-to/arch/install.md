---
title: "Установка Arch Linux"
type: how-to
tags: [linux, arch, install, partition, grub, chroot, base]
sources:
  original: "_inbox/01-linux/02-distro-specific/arch-linux/01-installation.md"
  wiki: "https://wiki.archlinux.org/title/Installation_guide"
related:
  - "[[linux/explanation/distributions]]"
  - "[[linux/how-to/arch/pacman-and-aur]]"
  - "[[linux/how-to/arch/maintenance]]"
---

# Установка Arch Linux

> **TL;DR:** Загрузка с ISO → разметка дисков → mount → pacstrap → chroot → настройка → GRUB → reboot.
> Ставить вручную хорошо, но есть archinstall.

Эта инструкция взята с arch wiki, здесь каждый шаг выполняется вручную. Для изучения можно попробовать, я предпочитаю **archinstall**

## Подготовка

1. Скачать ISO: [archlinux.org/download](https://archlinux.org/download/)
2. Записать на USB: `dd if=archlinux.iso of=/dev/sdX bs=4M status=progress`
3. Загрузиться с USB (UEFI рекомендуется)

## Шаг 1. Подключение к сети

```bash
# Проводная — обычно работает автоматически
ip a                       # проверить IP

# Wi-Fi
iwctl
> station wlan0 scan
> station wlan0 get-networks
> station wlan0 connect "SSID"
> exit

# Проверка
ping -c 3 archlinux.org
```

## Шаг 2. Разметка диска (UEFI + GPT)

```bash
# Определить диск
lsblk

# Разметка (пример для /dev/sda)
cfdisk /dev/sda
# Создать:
#   /dev/sda1  512M   EFI System
#   /dev/sda2  остаток  Linux filesystem

# Форматирование
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2

# Монтирование
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

## Шаг 3. Установка базовой системы

```bash
# Обновить зеркала (опционально, для скорости)
reflector --country "Germany,France" --protocol https --sort rate --save /etc/pacman.d/mirrorlist

# Установить базу
pacstrap -K /mnt base linux linux-firmware \
  base-devel vim networkmanager grub efibootmgr \
  sudo git curl wget htop man-db man-pages
```

## Шаг 4. Настройка системы

```bash
# Сгенерировать fstab
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab         # проверить!

# Войти в новую систему
arch-chroot /mnt

# --- Внутри chroot ---

# Timezone
ln -sf /usr/share/zoneinfo/Europe/Tallinn /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "ru_RU.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "archbox" > /etc/hostname

# Hosts
cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   archbox.localdomain archbox
EOF

# Root пароль
passwd

# Создать пользователя
useradd -m -G wheel -s /bin/bash username
passwd username

# Sudo для wheel
EDITOR=vim visudo
# Раскомментировать: %wheel ALL=(ALL) ALL
```

## Шаг 5. Загрузчик (GRUB)

```bash
# UEFI
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

## Шаг 6. Сеть

```bash
systemctl enable NetworkManager
```

## Шаг 7. Перезагрузка

```bash
exit                       # выйти из chroot
umount -R /mnt
reboot
```

## После установки

```bash
# Подключиться к Wi-Fi
nmcli device wifi connect "SSID" password "PASSWORD"

# Обновить систему
sudo pacman -Syu

# Установить AUR helper
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
```

Подробнее: [[linux/how-to/arch/pacman-and-aur]], [[linux/how-to/arch/maintenance]].

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Не загружается после установки | Проверить GRUB: `grub-install` + `grub-mkconfig` |
| Нет сети после reboot | `systemctl enable --now NetworkManager` забыли |
| `sudo: command not found` | Не установлен `sudo` или user не в группе `wheel` |
| `/boot` пустой | Забыли `mount /dev/sda1 /mnt/boot` перед `pacstrap` |
| Ошибка locale | `locale-gen` после редактирования `/etc/locale.gen` |