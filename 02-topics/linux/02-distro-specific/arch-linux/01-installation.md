# Arch Installation

## Overview

Установка Arch Linux. В современном Arch используется `archinstall` — официальный интерактивный установщик.

**Что вы узнаете:**
- Два способа установки: archinstall (рекомендуется) и manual (для обучения)
- Загрузка с USB и подготовка
- Пошаговая установка через archinstall
- Первая загрузка и конфигурация
- Manual способ (optional)

## Prerequisites

- USB флешка с ISO образом
- 20+ GB свободного места на диске
- Доступ в интернет (по кабелю или WiFi)
- Базовое понимание что такое диск и разделы

## Загрузка с USB

```bash
# 1. Вставьте флешку
# 2. Перезагрузитесь
# 3. При загрузке нажмите F12, Del, Esc или другую кнопку
# 4. Выберите USB в меню загрузки
# 5. Загрузится Arch Linux Live Environment (root доступ)
```

## Способ 1: Archinstall (Рекомендуется) ⭐

Официальный интерактивный установщик. Самый простой способ.

### Запуск

```bash
archinstall
# Просто набираете команду и следуете подсказкам
```

### Шаги установки

1. **Language** — выберите язык (English для начинающих)

2. **Mirrors** — выберите регион для зеркал (Russia, Europe, etc.)

3. **Localization** — региональные настройки
   - Keyboard layout: Russian (или ваш язык)
   - Locale: en_US.UTF-8

4. **Disk configuration** — выбор диска
   - Выберите диск (обычно `/dev/sda`)
   - Выберите filesystem (ext4 рекомендуется)
   - Best effort partitioning (автоматически разобьёт)

5. **Bootloader** — выбор загрузчика
   - grub (рекомендуется, простой)
   - systemd-boot (новый, минималистичный)

6. **Hostname** — имя компьютера
   - Введите имя (например: myarch)

7. **Root password** — пароль администратора
   - Введите сильный пароль

8. **User account** — создание пользователя
   - Username: ваше_имя
   - Password: сильный_пароль
   - ✓ Should this user be a superuser? — Yes (для sudo)

9. **Audio** — звук (optional)
   - pipewire (рекомендуется, современный)
   - pulseaudio (старый, но стабильный)
   - None (если не нужен звук)

10. **Network configuration** — сеть
    - networkmanager (рекомендуется, графический)
    - dhcp (автоматический, простой)

11. **Additional packages** — дополнительные пакеты (optional)
    - vim (редактор текста)
    - git (система контроля версий)
    - base-devel (инструменты разработки)

12. **Install** — начало установки
    - Нажмите Install
    - Ждите 5-10 минут

### После установки

```bash
# Перезагрузитесь
reboot

# Логиньтесь (может потребоваться несколько попыток для первого входа)
# Введите username и password

# Проверьте интернет
ping archlinux.org

# Обновите систему
sudo pacman -Syu
```

## Способ 2: Manual Installation (Для обучения)

Если хотите понять как работает Arch, или archinstall не работает.

### Подготовка

```bash
# Проверьте интернет
ping archlinux.org

# Если нет, подключитесь (для WiFi)
iwctl
station wlan0 scan
station wlan0 connect SSID
exit

# Синхронизируйте время
timedatectl set-ntp true
```

### Разбиение диска

```bash
lsblk                          # найдите диск (обычно /dev/sda)
cfdisk /dev/sda                # интерактивный редактор разделов

# Создайте разделы:
# /dev/sda1 - 512M - EFI System (тип ef)
# /dev/sda2 - остальное - Linux Filesystem
```

### Форматирование

```bash
mkfs.fat -F 32 /dev/sda1       # EFI раздел
mkfs.ext4 /dev/sda2            # корневой раздел (или btrfs)
```

### Монтирование

```bash
mount /dev/sda2 /mnt
mkdir -p /mnt/efi
mount /dev/sda1 /mnt/efi
```

### Установка базовой системы

```bash
pacstrap /mnt base linux linux-firmware grub efibootmgr vim

# Или для минимальной
pacstrap /mnt base linux linux-firmware
```

### Генерация fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab             # проверьте результат
```

### Chroot в новую систему

```bash
arch-chroot /mnt
# Теперь вы внутри новой системы!
```

### Конфигурация

```bash
# Время
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# Hostname
echo "myarch" > /etc/hostname

# Root пароль
passwd

# Пользователь
useradd -m -G wheel username
passwd username
EDITOR=nano visudo        # раскомментируйте %wheel
```

### Bootloader

```bash
# GRUB
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### Выход и загрузка

```bash
exit
umount -R /mnt
reboot
```

## Troubleshooting

### Archinstall не запускается

```bash
# Может быть не подключено интернет
ip link show
dhclient enp0s3                # получить IP

# Попробуйте снова
archinstall
```

### Система не загружается после установки

```bash
# Загрузитесь с USB
arch-chroot /mnt /dev/sda2     # смонтируйте корень
grub-mkconfig -o /boot/grub/grub.cfg
exit
reboot
```

### Забыли пароль

```bash
# Загрузитесь с USB
arch-chroot /mnt /dev/sda2
passwd username
exit
reboot
```

## Сравнение способов

| Аспект | Archinstall | Manual |
|--------|-------------|--------|
| **Сложность** | Низкая (меню) | Высокая (команды) |
| **Время** | 10-15 минут | 60-90 минут |
| **Обучение** | Не учит как работает | Учит как всё работает |
| **Надёжность** | 99% успеха | Зависит от опыта |
| **Рекомендуется** | ✓ ДА (всем) | ✗ Опытным только |

## Key Takeaways

- **Archinstall** — современный, интерактивный, рекомендуется
- **Manual** — если хотите понять как работает Arch
- **Оба способа приводят к одинаковой системе**
- **Arch не сложный, просто требует внимания к деталям**

## Related

- [[./02-pacman-guide.md|Pacman Guide]] — что делать после установки
- [[./04-maintenance.md|Maintenance]] — обслуживание системы
- [[../README.md|Arch Linux Index]] — полный индекс

## See Also

- [Official Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [Archinstall](https://wiki.archlinux.org/title/Archinstall) — официальный гайд по archinstall
- [Partitioning](https://wiki.archlinux.org/title/Partitioning) — разбиение дисков
