# Arch Troubleshooting

## Overview

Решение типичных проблем Arch Linux.

## Система не загружается

### Зависает на boot

```bash
# Перезагрузитесь с USB Live
# Смонтируйте систему:
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/efi
sudo arch-chroot /mnt

# Пересоздайте initramfs
mkinitcpio -P

# Проверьте bootloader
grub-mkconfig -o /boot/grub/grub.cfg
```

### GRUB не запускается

```bash
# Переустановите GRUB
sudo grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Black screen после загрузки

```bash
# Может быть GPU проблема
# Попробуйте ввести команды вслепую:
# Нажмите Ctrl+Alt+F2 для TTY
login
sudo pacman -S linux-lts      # откатитесь на LTS ядро
sudo reboot
```

## Нет интернета

### Ethernet не работает

```bash
ip link show                   # список интерфейсов
sudo ip link set enp0s3 up     # включить интерфейс
sudo dhclient enp0s3           # получить IP
```

### WiFi не работает

```bash
# Проверьте драйверы
lspci | grep -i network
lsmod | grep -i wifi

# Если нужны драйверы
sudo pacman -S broadcom-wl     # например для Broadcom

# Или используйте iwd
sudo pacman -S iwd
sudo systemctl start iwd
iwctl station wlan0 scan
iwctl station wlan0 connect SSID
```

## Пакет конфликтует

### Ошибка: "conflicting files"

```bash
# Способ 1: Переписать файлы
sudo pacman -S package --overwrite '*'

# Способ 2: Удалить конфликтующий пакет
sudo pacman -R conflicting-package
sudo pacman -S package

# Способ 3: Найти какой пакет содержит файл
pacman -Qo /path/to/conflicting/file
```

## Pacman проблемы

### Pacman зависает

```bash
# В другом терминале:
sudo pkill -9 pacman
sudo rm /var/lib/pacman/db.lck

# Потом попробуйте снова
sudo pacman -Syu
```

### Ошибка "could not open file"

```bash
# Пересоздайте базы данных
sudo pacman -Sc
sudo pacman -Syy
sudo pacman -Syu
```

## Нет звука

```bash
# Установите audio stack
sudo pacman -S pipewire wireplumber
sudo pacman -S pipewire-alsa   # совместимость с ALSA

# Или используйте PulseAudio (старее но стабильнее)
sudo pacman -S pulseaudio pulseaudio-alsa
```

## Нет видеодрайверов

### Intel GPU

```bash
sudo pacman -S xf86-video-intel
# или используйте встроенный моежель (рекомендуется)
```

### AMD GPU

```bash
sudo pacman -S xf86-video-amdgpu
```

### NVIDIA GPU

```bash
# Proprietary (требует много места)
sudo pacman -S nvidia

# Open source (новые NVIDIA)
sudo pacman -S xf86-video-nouveau
```

## Диск полностью заполнен

```bash
# Найдите что занимает место
du -sh /* | sort -rh | head -10
du -sh ~/.cache/*

# Очистьте
sudo pacman -Scc               # кэш pacman
sudo journalctl --vacuum=50M   # журналы
rm -rf ~/.cache/*              # пользовательский кэш
```

## Недостаточно памяти

```bash
# Уменьшите swap
sudo swapon --show
free -h

# Отключите ненужные сервисы
systemctl list-units --type=service --state=running
sudo systemctl disable service
```

## X/Display Server не запускается

```bash
# Проверьте логи
cat ~/.local/share/xorg/Xvfb.log
journalctl -u display-manager

# Переустановите X server
sudo pacman -S xorg xorg-xinit

# Или используйте Wayland (новее)
sudo pacman -S wayland
```

## Откат обновления

```bash
# Если обновление сломало систему
# Найдите старый пакет в кэше
ls -la /var/cache/pacman/pkg/package-*

# Откатитесь
sudo pacman -U /var/cache/pacman/pkg/package-old.pkg.tar.zst
```

## Kernel Panic

```bash
# Загрузитесь с USB Live

# Смонтируйте систему
sudo mount /dev/sda2 /mnt
sudo arch-chroot /mnt

# Откатитесь на предыдущее ядро
sudo pacman -S linux-lts

# Или обновите всю систему
sudo pacman -Syu

# Пересоздайте initramfs
mkinitcpio -P
```

## Key Takeaways

- **Arch Wiki первый помощник** — всегда идите туда
- **Перезагружайтесь с USB** — если система не загружается
- **Pacman может зависнуть** — `sudo rm /var/lib/pacman/db.lck`
- **Откаты возможны** — старые пакеты в `/var/cache/pacman/pkg`
- **Интернет критичен** — потеря интернета ломает многое

## Related

- [[./01-installation.md|Installation]] — если нужна переустановка
- [[./04-maintenance.md|Maintenance]] — профилактика
- [[../README.md|Arch Index]] — индекс

## See Also

- [General troubleshooting](https://wiki.archlinux.org/title/General_troubleshooting)
- [Boot process](https://wiki.archlinux.org/title/Arch_boot_process)
- [Systemd](https://wiki.archlinux.org/title/Systemd)
