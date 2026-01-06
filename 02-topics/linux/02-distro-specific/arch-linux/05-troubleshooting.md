---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# Решение проблем Arch Linux

## 🚨 ПОСЛЕ ОБНОВЛЕНИЯ СИСТЕМА НЕ ЗАГРУЖАЕТСЯ

### Симптомы
- Черный экран, не загружается
- "kernel panic"
- Ошибки initramfs

### РЕШЕНИЕ (через Live USB)

```bash
# 1. Загрузитесь с Arch Live USB
# 2. В приглашении root@archiso выполните:

# Смонтируйте систему
sudo mount /dev/sda2 /mnt    # Linux раздел (sda2 - ваш раздел)

# Вход в chroot
arch-chroot /mnt

# Переинсталлируйте kernel
sudo pacman -S linux         # стандартный kernel
# или
sudo pacman -S linux-lts     # LTS kernel (более стабильный)

# Пересоздайте initramfs
sudo mkinitcpio -P

# Переинсталлируйте GRUB
sudo grub-install /dev/sda
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Выход из chroot и перезагрузка
exit
sudo reboot
```

---

## ⚠️ ПОСЛЕ ЧАСТИЧНОГО ОБНОВЛЕНИЯ КОНФЛИКТЫ

### Симптомы
- "error: target not found"
- "error: could not satisfy dependencies"
- Не могу установить ничего

### РЕШЕНИЕ

```bash
# НИКОГДА не делайте частичное обновление!
# ВСЕГДА:
sudo pacman -Syu

# Если уже сломалось, спасение:
sudo pacman -Syu             # полное обновление

# Если конфликты файлов
sudo pacman -Syu --overwrite='*'

# Если совсем плохо
sudo pacman -Syy             # пересинхронизировать репозитории
sudo pacman -Syu             # попробовать снова
```

---

## 🔄 ОТКАТИТЬ ПАКЕТ НА СТАРУЮ ВЕРСИЮ

### Способ 1: Из кэша (если недавно обновлялся)

```bash
# Посмотреть что в кэше
ls /var/cache/pacman/pkg/ | grep package

# Установить старую версию
sudo pacman -U /var/cache/pacman/pkg/package-oldversion.tar.zst
```

### Способ 2: Arch Linux Archive

```bash
# Установить downgrade инструмент
sudo pacman -S downgrade

# Использовать
sudo downgrade package
# Выбрать версию из списка (стрелки, Enter)
```

### Способ 3: Вернуться на Btrfs снимок

```bash
# Если использовали Btrfs snapshots
sudo btrfs subvolume list /
sudo btrfs subvolume delete /.snapshots/current
sudo btrfs subvolume snapshot /.snapshots/backup-20260103 /

# Перезагрузитесь
sudo reboot
```

---

## 🔒 PACMAN ЗАБЛОКИРОВАН

### Симптомы
- "error: could not open lock file"
- pacman зависла при установке

### РЕШЕНИЕ

```bash
# Посмотреть есть ли другой pacman
ps aux | grep pacman

# Если есть другой процесс
sudo kill -9 PID              # убить процесс

# Удалить lock файл
sudo rm /var/lib/pacman/db.lck

# Попробовать снова
sudo pacman -Syu
```

---

## 🐢 МЕДЛЕННАЯ ЗАГРУЗКА

### Диагностика

```bash
# Сколько времени загрузка?
systemd-analyze

# Какие сервисы медленные?
systemd-analyze blame | head -20

# График зависимостей
systemd-analyze critical-chain
```

### РЕШЕНИЕ

```bash
# Посмотреть какие сервисы включены
systemctl list-unit-files --state=enabled | grep -E "network|bluetooth|cups"

# Отключить ненужные при загрузке
sudo systemctl disable network-manager        # если не нужен
sudo systemctl disable bluetooth              # если не нужен
sudo systemctl disable cups                   # если не печатаете

# Удалить ненужные пакеты
sudo pacman -Rns orphan-package

# Очистить кэш
sudo pacman -Sc
```

---

## 🔨 AUR ПАКЕТ НЕ КОМПИЛИРУЕТСЯ

### РЕШЕНИЕ

```bash
# 1. Посмотреть полную ошибку
yay -S package 2>&1 | tail -100

# 2. Очистить кэш сборки
cd ~/.cache/yay/package-name
rm -rf src pkg *.tar.zst

# 3. Проверить зависимости в PKGBUILD
cat PKGBUILD | grep depends

# 4. Установить зависимости вручную
yay -S dependency1 dependency2

# 5. Попробовать снова
yay -S package --rebuild

# 6. Если совсем не помогает
# Посмотрите в комментариях на AUR
# Может быть пакет временно сломан
```

---

## 🎬 VIDEO ДРАЙВЕР НЕ УСТАНОВИЛСЯ

### Диагностика

```bash
# Видит ли система GPU?
lspci | grep -i vga

# Какие драйверы установлены?
pacman -Qs video
pacman -Qs nvidia
pacman -Qs amdgpu
```

### NVIDIA

```bash
sudo pacman -S nvidia nvidia-utils
# или если используете DKMS
sudo pacman -S nvidia-dkms nvidia-utils

sudo reboot
```

### AMD

```bash
sudo pacman -S amdgpu xf86-video-amdgpu
# или просто
sudo pacman -S amdgpu

sudo reboot
```

### Intel

```bash
sudo pacman -S intel-media-driver libva-intel-driver
# или
sudo pacman -S xf86-video-intel

sudo reboot
```

---

## 🌐 ИНТЕРНЕТ НЕ РАБОТАЕТ

### Диагностика

```bash
# Есть ли сетевые интерфейсы?
ip link

# Есть ли IP адреса?
ip addr

# Можно ли пингануть?
ping 8.8.8.8

# Какой DNS?
cat /etc/resolv.conf
```

### РЕШЕНИЕ для кабеля

```bash
# Перезагрузить NetworkManager
sudo systemctl restart networkmanager

# Или systemd-networkd
sudo systemctl restart systemd-networkd
```

### РЕШЕНИЕ для WiFi

```bash
# Установить wifi инструменты
sudo pacman -S iw wpa_supplicant networkmanager

# Подключиться
nmtui                        # интерактивное меню
# или
iwctl                        # для iwd
```

---

## 💾 ФАЙЛОВАЯ СИСТЕМА READ-ONLY

### Симптомы
- "Read-only file system"
- Не могу ничего писать на диск

### РЕШЕНИЕ

```bash
# Проверить диск
sudo fsck -n /dev/sda2       # только проверка

# Если нужны исправления (через Live USB)
sudo umount /dev/sda2
sudo fsck -y /dev/sda2       # исправить

# Если срочно нужно писать
sudo mount -o remount,rw /

# Перезагрузиться для полного fix
sudo reboot
```

---

## 🔐 ЗАБЫЛИ ПАРОЛЬ

### РЕШЕНИЕ

```bash
# Загрузитесь с Live USB

# Смонтируйте систему
sudo mount /dev/sda2 /mnt

# Войдите в chroot
arch-chroot /mnt

# Установите новый пароль
passwd username              # для обычного пользователя
# или
passwd                       # для root

# Выход и перезагрузка
exit
sudo reboot
```

---

## 📋 ШПАРГАЛКА TROUBLESHOOTING

```bash
# Система не загружается (с Live USB)
arch-chroot /mnt
sudo pacman -S linux
sudo mkinitcpio -P
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Откатить пакет
sudo pacman -U /var/cache/pacman/pkg/package-old.tar.zst
sudo downgrade package

# pacman заблокирован
sudo rm /var/lib/pacman/db.lck

# Медленная загрузка
systemd-analyze blame

# Диагностика
journalctl -f                # логи
systemd-analyze              # время загрузки
ip addr                      # сетевые интерфейсы
```

---

## 🔗 ВАЖНЫЕ ССЫЛКИ

- **Arch Wiki**: https://wiki.archlinux.org
- **Arch Forum**: https://bbs.archlinux.org
- **Troubleshooting**: https://wiki.archlinux.org/title/Troubleshooting

---

## 🔗 ДАЛЬШЕ

[Ubuntu/Debian специфика](../ubuntu-debian/README.md)
