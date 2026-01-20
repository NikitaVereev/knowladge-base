---
title: "Установка и настройка Ansible"
description: "Инструкция по установке Ansible на Linux (Ubuntu, Arch), macOS и Windows (WSL)."
---

# Установка Ansible

Ansible устанавливается только на **Control Node** (ваш компьютер или сервер управления). На управляемые узлы ничего ставить не нужно, кроме Python.

## Способы установки

### Ubuntu / Debian
Используйте официальный PPA для получения свежей версии.

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

### Arch Linux
```bash
sudo pacman -S ansible
# Или из AUR (ansible-git) для dev-версии
```

### macOS
Через Homebrew:
```bash
brew install ansible
```

### Windows
Ansible **не работает** нативно в Windows. Используйте WSL2 (Windows Subsystem for Linux).
1. Установите WSL: `wsl --install`
2. Откройте терминал Ubuntu внутри Windows.
3. Следуйте инструкции для Ubuntu выше.

## Проверка установки
```bash
ansible --version
```

## Настройка (ansible.cfg)
В корне вашего проекта создайте файл `ansible.cfg`, чтобы переопределить настройки по умолчанию.

```ini
[defaults]
# Путь к файлу инвентаря
inventory = ./hosts.ini

# Отключить проверку SSH-ключей (удобно для тестов, небезопасно в проде)
host_key_checking = False

# Пользователь по умолчанию для подключения
remote_user = ubuntu

# Путь к приватному ключу
private_key_file = ~/.ssh/id_ed25519
```

## Связанные материалы
- [[devops/ansible/explanation/architecture|Архитектура Ansible]]
- [[devops/ansible/how-to/configure-ssh|Настройка SSH доступа]]
