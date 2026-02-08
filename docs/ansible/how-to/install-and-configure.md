---
title: "Установка и настройка Ansible"
type: how-to
tags: [ansible, install, pipx, ansible-cfg, pipelining, wsl, setup]
sources:
  docs: "https://docs.ansible.com/ansible/latest/installation_guide/index.html"
related:
  - "[[ansible/explanation/architecture]]"
  - "[[ansible/how-to/manage-inventory]]"
  - "[[ansible/tutorials/01-first-playbook]]"
---

# Установка и настройка Ansible

> **TL;DR:** `pipx install ansible` — рекомендуемый способ. Control Node — только Linux/macOS.
> ansible.cfg — рядом с playbook. Включить pipelining = ускорение в 3-5 раз.

Ansible устанавливается только на **Control Node**. На управляемые серверы ничего ставить не нужно, кроме Python и SSH.

## Установка

### Рекомендуемый: pipx (изолированная среда)

```bash
# Установить pipx
sudo pacman -S python-pipx          # Arch
sudo apt install pipx                # Ubuntu 23.04+
brew install pipx                    # macOS

pipx ensurepath                      # добавить в PATH

# Установить Ansible
pipx install ansible

# Дополнительные инструменты
pipx inject ansible ansible-lint argcomplete
```

### Системные пакеты

```bash
# Ubuntu / Debian (PPA для свежей версии)
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# RHEL / CentOS / Fedora
sudo dnf install epel-release
sudo dnf install ansible

# Arch Linux
sudo pacman -S ansible

# macOS
brew install ansible
```

### Windows (только через WSL2)

```powershell
wsl --install
```

Далее в Ubuntu-терминале — установка как для Linux.

## Настройка (ansible.cfg)

Ansible ищет конфигурацию: `$ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. Создайте файл в корне проекта.

```ini
[defaults]
# Путь к инвентарю
inventory = ./inventory/hosts.yml

# Формат вывода — читаемый YAML вместо JSON
stdout_callback = yaml
stderr_callback = yaml

# Отключить .retry файлы
retry_files_enabled = False

# Отключить проверку SSH Host Key (для динамических сред)
host_key_checking = False

# Сбор фактов: smart = кэшировать, explicit = только по запросу
gathering = smart

# Python-интерпретатор: не показывать предупреждения
interpreter_python = auto_silent

# Количество параллельных SSH-соединений (по умолчанию 5)
forks = 20

# Путь к коллекциям
collections_path = ./collections

[ssh_connection]
# КРИТИЧНО ДЛЯ СКОРОСТИ
# Pipelining = меньше SSH-операций, ускорение в 3-5 раз
# Требует отключённого requiretty в /etc/sudoers (обычно по умолчанию)
pipelining = True

# Переиспользование SSH-соединений (ControlPersist)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s

[privilege_escalation]
# become по умолчанию для всех playbook
become = False
become_method = sudo
become_ask_pass = False
```

## Проверка

```bash
# Версия Ansible
ansible --version

# Какой конфиг используется
ansible --version | head -1
# ansible [core 2.16.0]
#   config file = /home/user/project/ansible.cfg

# Ping всех хостов из inventory
ansible all -m ping

# Ping конкретной группы
ansible webservers -m ping
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Ansible из snap/apt без PPA | Старая версия (2.9 вместо 2.16) | Использовать pipx или PPA |
| Нет `pipelining = True` | Playbook выполняется медленно | Добавить в `[ssh_connection]` секцию ansible.cfg |
| `host_key_checking = True` | `UNREACHABLE! Host key verification failed` | Отключить или добавить ключи в known_hosts |
| Windows без WSL | `ansible: command not found` | Control Node — только Linux/macOS. Установить WSL2 |
| `forks = 5` (по умолчанию) | 100 хо