---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# Решение проблем Ubuntu/Debian

## 🚨 СИСТЕМА НЕ ЗАГРУЖАЕТСЯ (GRUB)

### Симптомы
- Черный экран с GRUB> приглашением
- Ошибки при загрузке
- Попадаете в GRUB shell

### РЕШЕНИЕ (с Live USB)

```bash
# 1. Загрузитесь с Live USB
# 2. Откройте терминал

# Смонтируйте Linux раздел
sudo mount /dev/sda3 /mnt       # sda3 - ваш Linux раздел
# (проверьте: sudo fdisk -l)

# Если используется LVM
sudo vgchange -ay               # активировать
sudo mount /dev/ubuntu-vg/root /mnt

# Смонтируйте системные папки
sudo mount /dev/sda1 /mnt/boot/efi    # EFI раздел
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys

# Войдите в chroot
sudo chroot /mnt

# Переинсталлируйте GRUB
sudo grub-install /dev/sda      # на MBR
# или для UEFI
sudo grub-install --efi-directory=/boot/efi /dev/sda

# Пересоздайте конфиг
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Выход и перезагрузка
exit
sudo reboot
```

---

## ⚠️ apt ЗАБЛОКИРОВАН

### Симптомы
- "E: Could not get lock"
- apt зависла при установке

### РЕШЕНИЕ

```bash
ps aux | grep apt                # ищем процесс
sudo killall -9 apt apt-get      # если зависла

sudo rm /var/lib/apt/lists/lock
sudo apt update
```

---

## 🔄 ОТКАТИТЬ ПАКЕТ НА СТАРУЮ ВЕРСИЮ

### Способ 1: Из кэша

```bash
ls /var/cache/apt/archives/ | grep package

sudo apt install /var/cache/apt/archives/package_oldversion.deb
```

### Способ 2: Указать версию

```bash
apt-cache policy package       # доступные версии

sudo apt install package=version
```

---

## 🐢 МЕДЛЕННАЯ ЗАГРУЗКА

### Диагностика

```bash
# Время загрузки
systemd-analyze

# Медленные сервисы
systemd-analyze blame | head -10

# График
systemd-analyze critical-chain
```

### РЕШЕНИЕ

```bash
# Отключить ненужные сервисы
sudo systemctl disable bluetooth
sudo systemctl disable cups
sudo systemctl disable avahi-daemon

# Удалить ненужные пакеты
sudo apt remove package-name
```

---

## 🌐 ИНТЕРНЕТ НЕ РАБОТАЕТ

### Диагностика

```bash
ip link                      # есть ли интерфейсы?
ip addr                      # есть ли IP?
ping 8.8.8.8                # можно ли пингануть?
cat /etc/resolv.conf        # DNS?
```

### РЕШЕНИЕ

```bash
# Перезагрузить сеть
sudo systemctl restart NetworkManager
sudo systemctl restart systemd-networkd

# или для кабеля
sudo systemctl restart networking
```

---

## 📦 PPA ПРОБЛЕМЫ

### "Signed by unknown key"

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys KEY_ID
sudo apt update
```

### PPA конфликтует

```bash
sudo add-apt-repository --remove ppa:user/ppa-name
sudo apt update
sudo ppa-purge ppa:user/ppa-name
```

---

## 💾 ДИСК READ-ONLY

### Симптомы
- "Read-only file system"
- Не могу писать на диск

### РЕШЕНИЕ

```bash
# Проверить диск (через Live USB)
sudo fsck -n /dev/sda3      # только проверка

# Если нужны исправления
sudo umount /dev/sda3
sudo fsck -y /dev/sda3      # исправить
```

---

## 🔐 ЗАБЫЛИ ПАРОЛЬ

### РЕШЕНИЕ

```bash
# С Live USB
sudo mount /dev/sda3 /mnt
sudo chroot /mnt

# Новый пароль для пользователя
passwd username
# или для root
passwd

# Выход
exit
sudo reboot
```

---

## 📋 ШПАРГАЛКА

```bash
# GRUB не работает (с Live USB)
sudo mount /dev/sda3 /mnt
sudo mount /dev/sda1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
sudo grub-install --efi-directory=/boot/efi /dev/sda
sudo grub-mkconfig -o /boot/grub/grub.cfg

# apt заблокирован
sudo rm /var/lib/apt/lists/lock
sudo apt update

# Откатить пакет
apt-cache policy package
sudo apt install package=version

# Медленная загрузка
systemd-analyze blame | head -10
```

---

## 🔗 ДАЛЬШЕ

[Arch Linux специфика](../arch-linux/README.md)
