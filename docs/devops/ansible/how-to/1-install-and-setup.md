---
title: "1 Установка и настройка Ansible"
description: "Инструкция по установке Ansible на Linux (Ubuntu, Arch), macOS и Windows (WSL)."
---

Ansible устанавливается только на **Control Node** (ваш компьютер или сервер управления). На управляемые узлы (Managed Nodes) ничего ставить не нужно, кроме Python.

## Способ 1: Системные пакетные менеджеры (System Packages)
Это самый простой метод, но он имеет минусы: версии в репозиториях (особенно Debian/RHEL) могут отставать на годы. Для `apt` мы используем PPA, чтобы обойти это ограничение.

### Ubuntu / Debian
Стандартные репозитории часто содержат очень старую версию. Рекомендуется подключить PPA.

```bash
# 1. Обновляем кэш и ставим утилиты для работы с репозиториями
sudo apt update
sudo apt install software-properties-common

# 2. Подключаем PPA (гарантирует свежую версию)
sudo add-apt-repository --yes --update ppa:ansible/ansible

# 3. Устанавливаем Ansible
sudo apt install ansible
```

### RHEL / CentOS / Fedora / AlmaLinux
В экосистеме Red Hat Ansible находится в репозитории EPEL (Extra Packages for Enterprise Linux).

```bash
# 1. Подключаем EPEL (если еще нет)
sudo dnf install epel-release

# 2. Устанавливаем Ansible
sudo dnf install ansible
```

### Arch Linux / Manjaro
В Arch Linux всегда "свежий" софт, поэтому никаких дополнительных репозиториев не нужно.

```bash
sudo pacman -S ansible
```

### macOS (Homebrew)
Стандарт де-факто для маков.

```bash
brew install ansible
```

---

## Способ 2: Изолированная среда (pipx) — Рекомендовано
Если вам нужна независимость от системных библиотек или возможность иметь несколько версий Ansible, используйте `pipx`. Он создает виртуальное окружение для утилиты, но делает команду доступной глобально.

1.  **Установите pipx** (если его нет):
    ```bash
    # Arch
    sudo pacman -S python-pipx
    # Ubuntu 23.04+
    sudo apt install pipx
    # macOS
    brew install pipx
    
    # Добавьте пути в PATH
    pipx ensurepath
    ```

2.  **Установите Ansible:**
    ```bash
    pipx install ansible
    ```

3.  **Добавьте полезные инструменты:**
    Например, линтер (`ansible-lint`) или библиотеку `argcomplete` для автодополнения.
    ```bash
    pipx inject ansible ansible-lint argcomplete
    ```

---

## Способ 3: Windows (Только через WSL2)
Нативная установка на Windows не поддерживается и не работает корректно. Используйте **WSL2** (подсистему Linux).

1.  Запустите PowerShell от имени администратора:
    ```powershell
    wsl --install
    ```
2.  После перезагрузки откройте "Ubuntu" из меню "Пуск".
3.  Выполните инструкции для **Ubuntu** (см. выше Способ 1 или 2) внутри этого терминала.

---

## Настройка (ansible.cfg)
Ansible ищет конфигурацию в текущей папке запуска. Создайте файл `ansible.cfg` в корне вашего репозитория.

```ini
[defaults]
# Указываем путь к инвентарю по умолчанию
inventory = ./inventory/hosts.yml

# Отключаем проверку Host-ключей (удобно для динамических сред/облаков)
# В High-Security средах лучше включить и управлять known_hosts
host_key_checking = False

# !ВАЖНО! Меняем формат вывода с JSON на YAML
# Это делает логи в терминале читаемыми для человека
stdout_callback = yaml
stderr_callback = yaml

# Отключаем создание .retry файлов (мусор в репозитории)
retry_files_enabled = False

# Ускоряем работу: не собирать факты, если они не нужны (explicit)
# Или используйте 'smart' для кэширования
gathering = smart

# Подавляем предупреждения о python interpreter
interpreter_python = auto_silent

# Указываем пользователю, где искать коллекции (если ставим локально в проект)
collections_path = ./collections

[ssh_connection]
# !КРИТИЧНО ДЛЯ СКОРОСТИ!
# Pipelining уменьшает кол-во SSH-соединений, выполняя скрипты через pipe.
# Ускоряет выполнение плейбуков в 3-5 раз.
# Требует отключенного 'requiretty' в /etc/sudoers на целевых машинах (обычно дефолт).
pipelining = True

# Переиспользование SSH-соединений (ControlPersist)
# Держит сессию открытой 60 сек после команды
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

## Проверка

### 1. Проверка установки (Локально)
Убедитесь, что Ansible установлен корректно и версия актуальна:
```bash
ansible --version
```
*Вывод покажет версию, путь к конфигу и версию Python.*

### 2. Проверка связности (Ping)
После того как вы настроите инвентарь (файл `hosts`), проверьте, может ли Ansible достучаться до серверов. Это делается модулем `ping`.
> **Важно:** Это не ICMP-пинг (как в сети), а попытка зайти по SSH, исполнить Python-код и вернуть ответ `pong`.

```bash
ansible all -m ping
```
*Если видите зеленый `SUCCESS` и `"ping": "pong"` — всё работает.*
