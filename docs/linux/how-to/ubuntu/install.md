---
title: "Установка Ubuntu"
type: how-to
tags: [linux, ubuntu, install, gui, partition, lts]
sources:
  original: "_inbox/01-linux/02-distro-specific/ubuntu-linux/01-installation.md"
related:
  - "[[linux/explanation/distributions]]"
  - "[[linux/how-to/ubuntu/apt-and-ppa]]"
  - "[[linux/how-to/ubuntu/maintenance]]"
---

# Установка Ubuntu

> **TL;DR:** Скачать ISO → записать на USB (Rufus/dd) → загрузиться → графический установщик.
> Рекомендуется LTS-версия (24.04). Minimal installation для серверов.

## Подготовка

1. Скачать ISO: [ubuntu.com/download](https://ubuntu.com/download)
   - **Desktop:** Ubuntu 24.04 LTS (с GUI)
   - **Server:** Ubuntu Server 24.04 LTS (без GUI)
2. Записать на USB:
   ```bash
   # Linux
   sudo dd if=ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress
   # Или использовать Ventoy, Rufus (Windows), balenaEtcher
   ```
3. Загрузиться с USB (F12/Del/Esc при загрузке)

## Установка (Desktop)

### 1. Язык и раскладка
Выберите язык → «Install Ubuntu».

### 2. Тип установки
- **Normal installation** — браузер, офис, утилиты (рекомендуется)
- **Minimal installation** — только базовые утилиты

Отметьте: «Download updates while installing» + «Install third-party software» (драйверы, кодеки).

### 3. Разметка диска
- **Erase disk and install Ubuntu** — весь диск (самый простой вариант)
- **Something else** — ручная разметка:

| Раздел | Размер | Тип | Точка монтирования |
|--------|--------|-----|-------------------|
| EFI | 512 MB | EFI System Partition | `/boot/efi` |
| Root | 30+ GB | ext4 | `/` |
| Home | остаток | ext4 | `/home` |
| Swap | 2-4 GB | swap | — |

### 4. Пользователь
Имя, имя компьютера, пароль. Этот пользователь автоматически получает sudo.

### 5. Перезагрузка
Извлеките USB → перезагрузка.

## После установки

```bash
# Обновить всё
sudo apt update && sudo apt upgrade

# Установить полезные утилиты
sudo apt install git vim curl wget htop tree build-essential

# Установить проприетарные драйверы
sudo ubuntu-drivers autoinstall

# Настроить SSH (если сервер)
sudo apt install openssh-server
sudo systemctl enable ssh
```

## Ubuntu Server (без GUI)

```bash
# Установщик текстовый (Subiquity), но тоже пошаговый:
# 1. Язык → 2. Сеть → 3. Диск → 4. Пользователь → 5. SSH

# После установки:
sudo apt update && sudo apt upgrade
sudo apt install vim htop tmux
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| USB не загружается | Проверить UEFI/Legacy в BIOS, отключить Secure Boot |
| Не видит Wi-Fi при установке | Подключить Ethernet. Драйверы Wi-Fi установятся после |
| Dual-boot с Windows | Выбрать «Something else», не трогать Windows-разделы |
| Мало места после установки | Minimal installation, отдельный `/home` |