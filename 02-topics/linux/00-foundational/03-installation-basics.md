---
created: 2026-01-04
updated: 2026-01-04
type: reference
---

# Установка Linux

## Способы установки

### VirtualBox (виртуальная машина)

Ядро работает как приложение внутри вашей ОС.

**Плюсы:** Безопасно, легко удалить, попробуете сначала  
**Минусы:** Медленнее, занимает RAM

**Установка:**

1. Скачайте VirtualBox: https://virtualbox.org
2. Скачайте Ubuntu ISO: https://ubuntu.com/download/desktop
3. Создайте VM, выберите ISO
4. VirtualBox запустит установку
5. Следуйте инструкциям

### Live USB (с флешки)

Linux работает с USB, ничего на диск не записывается.

**Плюсы:** Тестируете на реальном ПК  
**Минусы:** Медленно, всё теряется при выключении

**Создание:**

На Windows: используйте Rufus  
На macOS/Linux:
```bash
# Узнайте имя USB
diskutil list

# Размонтируйте
diskutil unmountDisk /dev/diskX

# Запишите
sudo dd if=~/Downloads/ubuntu.iso of=/dev/rdiskX bs=4m

# Эжект
diskutil eject /dev/diskX
```

### Dual-boot (Windows + Linux)

Оба ОС на одном компьютере.

**Плюсы:** Реальная производительность  
**Минусы:** Нужно разделить диск, ⚠️ может поломать Windows

**Требуется:** Резервная копия данных!

### Bare metal (только Linux)

Удаляется Windows, устанавливается только Linux.

**Плюсы:** Полная производительность  
**Минусы:** Необратимо

---

## Загрузка с USB при старте

При включении компьютера нажимайте:

| ПК | Кнопка |
|----|--------|
| Lenovo, Dell | F12 |
| HP | Esc |
| ASUS, Gigabyte | Del |
| Mac | Option |

Выберите USB в меню загрузки.

---

## После установки

### Обновиться

```bash
# Debian/Ubuntu
sudo apt update
sudo apt upgrade -y

# Fedora/RHEL
sudo dnf upgrade -y

# Arch
sudo pacman -Syu
```

### Установить нужное

```bash
# Debian/Ubuntu
sudo apt install curl git vim htop

# Fedora
sudo dnf install curl git vim htop

# Arch
sudo pacman -S curl git vim htop
```

### Узнать информацию о системе

```bash
# ОС и версия
cat /etc/os-release

# Kernel
uname -r

# Дисковое пространство
df -h

# Оперативная память
free -h

# Процессор
nproc
```

---

## Проблемы и решения

### Чёрный экран после загрузки

Нажмите **Ctrl+Alt+T** — должен открыться терминал. Linux работает, просто GUI не загрузился.

**Решение:** Установите видеодрайвер:
```bash
# NVIDIA
sudo apt install nvidia-driver-550

# AMD
sudo apt install amdgpu-dkms

# Intel (обычно работает из коробки)
```

### Нет интернета

```bash
# Если интерфейс есть но нет IP
sudo dhclient

# Или перезагрузитесь
sudo reboot
```

### Нет звука

```bash
# Debian/Ubuntu
sudo apt install alsa-utils

# Включите громкость
alsamixer  # нажмите стрелки, Enter для включения
```

### При загрузке с USB ничего не происходит

- Попробуйте другой USB порт
- Создайте USB заново через Rufus/dd
- В BIOS отключите Secure Boot

### Зависает при установке

- Попробуйте другой вариант графики: `nomodeset`
- В меню загрузки добавьте опцию: нажмите E, добавьте `nomodeset` перед `quiet`

### Ошибка при apt update

```bash
# Обновите список пакетов с нуля
sudo rm /var/lib/apt/lists/* -vf
sudo apt update
```

### Забыл пароль root

При загрузке нажмите E в меню GRUB, найдите строку с `linux` и добавьте `init=/bin/bash` в конце.

---

## Важные директории

```
/home/user/        домашняя папка пользователя
/root/             домашняя папка root
/etc/              конфигурационные файлы
/var/log/          логи системы
/tmp/              временные файлы
/opt/              программы третьих сторон
```

---

## Дальше

[Основные концепции](../01-core-concepts/README.md)
