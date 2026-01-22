---
title: "1 Установка и настройка Ansible"
description: "Инструкция по установке Ansible на Linux (Ubuntu, Arch), macOS и Windows (WSL)."
---

Ansible устанавливается только на **Control Node** (ваш компьютер или сервер управления). На управляемые узлы (Managed Nodes) ничего ставить не нужно, кроме Python.

## Способы установки

### Ubuntu / Debian
Рекомендуется использовать PPA для получения свежей версии.

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

### Arch
Через pacman
```bash
sudo pacman -S ansible
```

### macOS
Через Homebrew:
```bash
brew install ansible
```

### Windows
Ansible **не работает** нативно в Windows. Используйте WSL2 (Windows Subsystem for Linux).
1. Установите WSL: `wsl --install`
2. Перезагрузитесь и откройте терминал Ubuntu.
3. Следуйте инструкции для Ubuntu выше.

### Python (pip)
Универсальный способ (рекомендуется использовать в venv):
```bash
pip install ansible
```

## Настройка (ansible.cfg)
В корне вашего проекта создайте файл `ansible.cfg`, чтобы переопределить настройки по умолчанию.

```ini
[defaults]
# Путь к файлу инвентаря по умолчанию
inventory = ./hosts.yaml

# Отключить проверку SSH-ключей (удобно для Vagrant/Test, небезопасно в проде)
host_key_checking = False

# Пользователь по умолчанию для подключения
remote_user = ubuntu

# Путь к приватному ключу
private_key_file = ~/.ssh/id_ed25519

# Отключить создание .retry файлов
retry_files_enabled = False
```
