---
title: "Установка VirtualBox"
description: "Пошаговая инструкция по установке VirtualBox и созданию виртуальной машины с Ubuntu 24.04 Server вручную."
---


Это руководство описывает процесс "ручной" сборки виртуальной машины. Если вам нужна автоматизация, используйте [[tools/environments/vagrant/tutorials/getting-started|Vagrant]].

## Предварительные требования

- **ISO образ:** Ubuntu 24.04 LTS Server (скачан с официального сайта).
- **Ресурсы:** Минимум 2 ГБ RAM и 20 ГБ свободного места на диске.
- **Хост-система:** В примере используется Arch Linux, но шаги 5-9 универсальны.

## Arch linux

Если у вас Ubuntu/Debian/macOS/Windows, пропустите этот шаг и установите VirtualBox через официальный инсталлятор или пакетный менеджер.

1. **Проверка ядра:**
   Arch Linux требует совместимости модулей ядра.
   ```bash
   uname -r
   ```
   *Результат:* Рекомендуется стандартное ядро `linux`. Для `linux-lts` или `linux-zen` потребуются соответствующие пакеты заголовков.

2. **Установка пакетов:**
   ```bash
   sudo pacman -Syu
   sudo pacman -S linux-headers virtualbox virtualbox-host-modules-arch
   ```
   * `linux-headers`: необходимы для сборки модулей ядра.
   * `virtualbox-host-modules-arch`: готовые модули для стандартного ядра Arch.

3. **Загрузка модулей ядра:**
   ```bash
   sudo modprobe vboxdrv
   lsmod | grep vbox
   ```
   *Ожидаемый вывод:* наличие `vboxdrv`, `vboxnetadp`, `vboxnetflt`.
   Если модули не стартуют: `sudo /sbin/vboxconfig`.

4. **Настройка прав доступа:**
   Чтобы пробрасывать USB устройства в VM:
   ```bash
   sudo usermod -aG vboxusers $USER
   # Требуется перелогиниться, чтобы группа применилась
   newgrp vboxusers
   ```

## Ubuntu / Debian

```bash
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack
```

Проверка:
```bash
VBoxManage --version
```

## macOS

Рекомендуемый способ через Homebrew:

```bash
brew install --cask virtualbox
```

Если установка не удалась из-за разрешений безопасности, перейдите в **System Settings → Privacy & Security** и разрешите запуск ПО от Oracle.

Проверка установки:
```bash
VBoxManage --version
```

Проверка:
```bash
VBoxManage --version
```

## Windows

Рекомендуемый способ через Chocolatey:

```powershell
choco install virtualbox
```

Альтернативно скачайте инсталлятор с [официального сайта](https://www.virtualbox.org/wiki/Downloads).

## Создание виртуальной машины

1. **Базовая конфигурация:**
   - Нажмите **New** (Создать).
   - **Name:** `Docker-Lab`
   - **Type:** Linux
   - **Version:** Ubuntu (64-bit)
   - **Memory:** 2048 MB (минимум для комфортной работы без GUI, с GUI — 4096 MB).
   - **Disk:** VDI, Dynamically allocated (Динамический), 20 GB.

2. **Подключение ISO:**
   - Settings (Настройки) -> Storage (Носители).
   - Controller: IDE -> Empty (Пусто).
   - Нажмите значок диска справа -> Choose a disk file -> Выберите `ubuntu-24.04-live-server-amd64.iso`.

3. **Настройка сети (Port Forwarding):**
   Чтобы подключаться по SSH с хоста, пробросим порт.
   - Settings -> Network -> Adapter 1 (NAT).
   - Раскройте **Advanced** -> **Port Forwarding**.
   - Добавьте правило:
     - **Protocol:** TCP
     - **Host IP:** 127.0.0.1 (безопаснее, чтобы не светить порт в локальную сеть)
     - **Host Port:** 2222
     - **Guest Port:** 22

## Установка ОС

1. Запустите VM (**Start**).
2. Следуйте инструкциям инсталлятора Ubuntu.
3. **Важно:** На экране выбора софта ("Profile Setup" или "SSH Setup") обязательно отметьте **Install OpenSSH server**. (Используйте `Space` для выбора, `Tab` для перехода).
4. Завершите установку и перезагрузитесь (извлеките ISO, если VirtualBox не сделал это автоматически).

## Проверка подключения

После загрузки VM (появится приглашение login), откройте терминал **на хост-машине**:

```bash
ssh username@127.0.0.1 -p 2222
```

Где `username` — пользователь, которого вы создали при установке Ubuntu.

## Связанные материалы

- [[tools/environments/vagrant/tutorials/getting-started]] — автоматическое создание VM.
- [[tools/ssh/how-to/generate-keys]] — настройка входа по ключам вместо пароля.
