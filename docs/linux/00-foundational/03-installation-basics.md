# Installation Basics

## Overview

Установка Linux - ваш первый реальный контакт с системой. В этом разделе мы расскажем о разных способах установки и как выбрать подходящий для ваших нужд.

**Что вы узнаете:**
- Четыре способа установки (VirtualBox, Live USB, dual-boot, bare metal)
- Как создать загрузочный USB
- Как загрузиться с USB при старте компьютера
- Команды после установки (обновление, установка ПО)
- Решение типичных проблем при установке

## Prerequisites

Перед этим разделом нужно:
- Понимание что такое дистрибутив ([[02-distributions-guide|Choosing a Distribution]])
- Выбранный дистрибутив (скачанный ISO образ)
- Компьютер или виртуальная машина

## Installation Methods

### 1. VirtualBox (Virtual Machine)

**Суть:** Установить Linux как приложение внутри вашей текущей ОС.

Linux работает как программа, управляется хост-ОС (Windows, macOS, Linux).

#### Плюсы ✓

- **Безопасно** — не трогает реальный компьютер
- **Легко удалить** — просто удалите папку
- **Попробовать сначала** — идеально для новичков
- **Несколько систем** — установите несколько дистро одновременно

#### Минусы ✗

- **Медленнее** — эмуляция замедляет процесс
- **Требует RAM** — виртуальная машина занимает оперативную память
- **Не реальная производительность** — графика, видео могут тормозить

#### How to Install

1. **Скачайте VirtualBox:**
   - https://virtualbox.org/download
   - Выберите для вашей ОС (Windows, macOS, Linux)
   - Установите как обычную программу

2. **Скачайте ISO образ дистрибутива:**
   - Ubuntu: https://ubuntu.com/download/desktop
   - Fedora: https://fedoraproject.org/download
   - Debian: https://debian.org/download
   - Arch: https://archlinux.org/download

3. **Создайте виртуальную машину:**
   ```
   File → New
   Name: Ubuntu 24.04
   Memory: 4096 MB (для нормальной работы)
   Disk: 30 GB (для системы + программ)
   ```

4. **Выберите ISO при первой загрузке:**
   ```
   Settings → Storage → Empty (CD icon)
   → Выберите скачанный ISO
   ```

5. **Запустите машину** и следуйте инструкциям установщика

#### Требования к ПК

```
Минимум:
- 4 GB RAM всего (2 GB для VM + 2 GB для хоста)
- 40 GB SSD свободного места

Рекомендуется:
- 8+ GB RAM
- 100 GB SSD свободного места
```

### 2. Live USB (Boot from Flash Drive)

**Суть:** Создать загрузочный USB и запустить/установить с него.

Linux работает с USB, ничего на диск не записывается.

#### Плюсы ✓

- **На реальном ПК** — тестируете производительность
- **Не деструктивно** — данные на диске не трогаются
- **Портативно** — USB можно использовать на разных ПК
- **Нет установки диска** — просто загрузитесь с USB

#### Минусы ✗

- **Медленно** — USB медленнее диска
- **Всё теряется при выключении** — нет persistent storage
- **Для установки нужен диск** — всё равно нужно установить на реальный диск

#### How to Create USB

##### На Windows: Используйте Rufus

1. Скачайте [Rufus](https://rufus.ie)
2. Вставьте USB флешку
3. Откройте Rufus, выберите:
   - Device: ваш USB
   - Boot selection: выберите скачанный ISO
   - Partition scheme: GPT (для новых ПК) или MBR (для старых)
4. Нажмите START и подождите

##### На macOS/Linux: Используйте dd

```bash
# 1. Узнайте имя USB
diskutil list

# Вывод будет примерно:
# /dev/disk0 (internal)
# /dev/disk1 (external, 16 GB) ← ваш USB

# 2. Размонтируйте (замените diskX на ваш номер)
diskutil unmountDisk /dev/diskX

# 3. Запишите ISO (осторожно с выбором диска!)
sudo dd if=~/Downloads/ubuntu.iso of=/dev/rdiskX bs=4m
# wait for completion...

# 4. Эжектируйте
diskutil eject /dev/diskX
```

⚠️ **ВАЖНО:** Проверьте правильность `/dev/diskX` - ошибка может стереть не тот диск!

### 3. Dual Boot (Windows + Linux)

**Суть:** Оба ОС на одном компьютере, выбор при загрузке.

#### Плюсы ✓

- **Реальная производительность** — нет эмуляции
- **Можно пользоваться Windows** — когда нужна Windows

#### Минусы ✗

- ⚠️ **Опасно** — можно повредить Windows или данные
- **Сложнее** — нужно разбить диск
- **Занимает место** — диск раздвоится

#### Требования

- ✓ **Резервная копия** — обязательно! (важные данные сохраните)
- ✓ **Место на диске** — минимум 40 GB свободного места
- ✓ **Время** — может занять 2-3 часа
- ✓ **BIOS доступ** — сможете зайти в BIOS/UEFI

#### Процесс (общий)

1. Делайте резервную копию Windows
2. Создайте Live USB
3. Загрузитесь с USB
4. Во время установки выберите "Something else" (Custom)
5. Разделите диск:
   - Windows: примерно половина
   - Linux: другая половина
6. Установите Linux
7. После перезагрузки будет выбор ОС

**Подробнее:** Инструкции специфичны для каждого дистрибутива и установщика - при установке вас просят выбрать что делать с диском.

### 4. Bare Metal (Linux Only)

**Суть:** Удалить все, установить только Linux.

#### Плюсы ✓

- **Полная производительность** — ничего не замедляет
- **Полный контроль** — вся система ваша

#### Минусы ✗

- ⚠️ **Необратимо** — Windows удаляется полностью
- ⚠️ **Опасно** — потеря всех данных

#### Требования

- ✓ **Резервная копия** — все важные данные должны быть сохранены
- ✓ **Уверенность** — знайте что вы делаете

## Booting from USB at Startup

Чтобы загрузиться с USB вместо жесткого диска, нужно выбрать в меню загрузки.

### Кнопки для разных производителей

| Производитель | Кнопка |
|---------------|--------|
| **Lenovo** | F12 или Fn+F12 |
| **Dell** | F12 |
| **HP** | Esc или F9 |
| **ASUS** | Del или F2 |
| **Gigabyte** | Del |
| **MSI** | Del |
| **Mac** | Option (⌥) |
| **Acer** | Del |

### Процесс

1. Вставьте USB флешку
2. Выключите компьютер
3. Включите и сразу нажимайте кнопку (смотрите таблицу выше)
4. В меню загрузки выберите ваш USB
5. Linux должен загрузиться

Если ничего не работает:
- Попробуйте другую кнопку из таблицы
- В BIOS отключите "Secure Boot"
- Убедитесь что USB правильно создан

## After Installation

### Update the System

После свежей установки обновите систему:

**Debian/Ubuntu:**
```bash
sudo apt update
sudo apt upgrade -y
sudo apt full-upgrade -y  # более агрессивное обновление
```

**Fedora/RHEL:**
```bash
sudo dnf upgrade -y
```

**Arch:**
```bash
sudo pacman -Syu
```

### Install Basic Tools

Установите полезные утилиты:

**Debian/Ubuntu:**
```bash
sudo apt install \
  curl \
  git \
  vim \
  htop \
  neofetch
```

**Fedora:**
```bash
sudo dnf install \
  curl \
  git \
  vim \
  htop \
  neofetch
```

**Arch:**
```bash
sudo pacman -S \
  curl \
  git \
  vim \
  htop \
  neofetch
```

### Check System Information

Проверьте что всё установилось правильно:

```bash
# ОС и версия
cat /etc/os-release

# Linux kernel версия
uname -r

# Дисковое пространство
df -h

# Оперативная память
free -h

# Количество ядер процессора
nproc

# Красивая информация (если установлен neofetch)
neofetch
```

## Troubleshooting

### "Black screen after boot"

Графический интерфейс не загрузился.

**Решение:**
```bash
# Попробуйте Ctrl+Alt+T чтобы открыть терминал
# Если терминал открылся, то ОС работает!

# Установите графические драйверы

# NVIDIA:
sudo apt install nvidia-driver-550

# AMD (Intel обычно работает из коробки):
sudo apt install amdgpu-dkms

# После установки перезагрузитесь:
sudo reboot
```

### "No internet connection"

Нет доступа в интернет.

**Решение:**
```bash
# Если интерфейс есть но IP не получена:
sudo dhclient

# Или попробуйте перезагрузиться:
sudo reboot

# Проверьте что вы подключены:
ip addr show
# должна быть строка с IP адресом
```

### "No sound"

Нет звука.

**Решение:**
```bash
# Debian/Ubuntu: установите аудио утилиты
sudo apt install alsa-utils pulseaudio

# Откройте микшер
alsamixer

# Используйте стрелки ↑↓ для увеличения громкости
# Нажмите 'M' чтобы включить/выключить звук
# Нажмите 'Q' чтобы выйти
```

### "USB boot doesn't work"

Ничего не происходит при загрузке с USB.

**Решение:**
1. Попробуйте другой USB порт
2. Создайте USB заново с помощью Rufus или dd
3. В BIOS отключите "Secure Boot"
4. Убедитесь что USB является первым в порядке загрузки

### "Hangs during installation"

Установка зависает или черный экран.

**Решение:**
Попробуйте отключить графику:
1. На экране загрузки нажмите 'E' (в GRUB)
2. Найдите строку `linux` и добавьте `nomodeset` перед `quiet`
3. Нажмите Ctrl+X для загрузки
4. Продолжите установку

### "Forgot root password"

Забыли пароль root.

**Решение:**
1. При загрузке нажмите 'E' в меню GRUB
2. Найдите строку начинающуюся с `linux`
3. Перейдите в конец строки (End)
4. Добавьте: `init=/bin/bash`
5. Нажмите Ctrl+X
6. Теперь вы в shell как root
7. Установите новый пароль:
   ```bash
   passwd username
   ```
8. Перезагрузитесь:
   ```bash
   reboot
   ```

### "apt/dnf error after update"

Ошибка при обновлении пакетов.

**Решение (Debian/Ubuntu):**
```bash
# Очистите кэш пакетов
sudo rm /var/lib/apt/lists/* -vf
sudo apt update
```

**Решение (Fedora):**
```bash
sudo dnf clean all
sudo dnf update
```

## Important Directories

После установки полезно знать основные директории:

```
/home/username/         ваша домашняя папка (все ваши файлы)
/root/                  домашняя папка администратора
/etc/                   конфигурационные файлы системы
/var/log/               логи системы (когда что-то падает)
/tmp/                   временные файлы (удаляются при перезагрузке)
/opt/                   программы третьих сторон
/usr/local/             локально установленное ПО
```

## Key Takeaways

- **VirtualBox** — для новичков, для попробовать
- **Live USB** — для тестирования перед установкой
- **Dual-boot** — если нужна Windows
- **Bare metal** — если только Linux
- **Всегда делайте backup** — перед установкой сохраните важные данные
- **После установки обновитесь** — важные security patches

## Related

Следующий шаг:
- [[docs/linux/01-core-concepts/README|Core Concepts]] — фундаментальные знания о Linux

Предыдущий раздел:
- [[02-distributions-guide|Choosing a Distribution]] — как выбрать дистрибутив

Контекст:
- [[docs/linux/00-foundational/README|Foundational Index]] — полный индекс этого раздела

## See Also

Утилиты:
- [Rufus](https://rufus.ie) — создание USB на Windows
- [VirtualBox](https://virtualbox.org) — виртуализация
- [Etcher](https://www.balena.io/etcher) — альтернатива Rufus
- [QEMU](https://www.qemu.org) — альтернатива VirtualBox

Дистрибутив-специфичные инструкции:
- [Ubuntu Installation Guide](https://ubuntu.com/tutorials/install-ubuntu-desktop)
- [Fedora Installation Guide](https://docs.fedoraproject.org/en-US/fedora/latest/install-guide/)
- [Debian Installation Guide](https://www.debian.org/doc/manuals/debian-handbook/ch04.en.html)

Общие ресурсы:
- `man grub` — информация о загрузчике
- `man fdisk` — информация о разбиении диска
