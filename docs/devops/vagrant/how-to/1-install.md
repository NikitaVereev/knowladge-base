---
title: "1 Установка Vagrant"
description: "Пошаговая инструкция по установке Vagrant и VirtualBox на основные ОС."
---

Для работы Vagrant требуется два компонента: сам Vagrant (утилита управления) и Гипервизор (среда виртуализации). Мы будем использовать VirtualBox как самый стабильный бесплатный вариант.

## Шаг 1: Установка VirtualBox

Скачайте и установите VirtualBox для вашей платформы с [официального сайта](https://www.virtualbox.org/wiki/Downloads).
*Важно: На macOS разрешите установку расширений ядра (Kernel Extensions) в настройках безопасности, если потребуется.*

## Шаг 2: Установка Vagrant

### macOS
Используйте Homebrew:
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vagrant
```

### Windows
Используйте Chocolatey:
```powershell
choco install vagrant
```
Или скачайте MSI установщик с [сайта HashiCorp](https://developer.hashicorp.com/vagrant/install). После установки **обязательно перезагрузитесь**.

### Linux (Ubuntu/Debian)
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

### Linux (Arch)
```bash
yay -S vagrant
```

## Шаг 3: Проверка

Откройте терминал и выполните:
```bash
vagrant --version
```
Успешный вывод: `Vagrant 2.4.x`
