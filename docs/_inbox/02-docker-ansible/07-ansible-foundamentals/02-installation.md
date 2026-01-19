---
title: 02 Установка и Конфигурация
---

---

## Установка по ОС

**Arch Linux:**
```bash
# Обновить систему
sudo pacman -Syu

# Установить Ansible через pacman
sudo pacman -S ansible

# ИЛИ через pip (свежая версия)
sudo pacman -S python-pip
pip install ansible

# ИЛИ из AUR (последняя dev версия)
yay -S ansible
# или paru -S ansible

# Проверка
ansible --version
```

**Ubuntu 24.04:**
```bash
# Метод 1: PPA (рекомендуется)
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# Метод 2: pip
sudo apt install python3-pip
pip install ansible

# Проверка
ansible --version
```

**macOS:**
```bash
# Через Homebrew
brew install ansible
brew upgrade ansible

# ИЛИ через pip
pip3 install ansible

# Проверка
ansible --version
```

**Windows (WSL2 рекомендуется):**
```powershell
# PowerShell от администратора
wsl --install -d Ubuntu-24.04

# В WSL терминале
wsl
sudo apt update
sudo apt install ansible
ansible --version
```

---

## Проверка Установки

```bash
# Версия Ansible
ansible --version
# Output: ansible [core 2.16.0]

# Список всех модулей
ansible-doc -l | wc -l
# Output: ~5000 модулей

# Документация по модулю
ansible-doc apt | head -50

# Краткая справка
ansible-doc -s apt
```

---

## SSH Ключи

**Почему SSH ключи?**
- Безопаснее чем пароли
- Не нужно вводить пароль при каждом запуске
- Возможность автоматизации

**Генерировать ED25519 ключ (рекомендуется):**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
# -t ed25519: современный алгоритм
# -f: путь к файлу
# -N "": без пароля (можно добавить пароль)

ls -la ~/.ssh/
# id_ed25519      (приватный ключ - СЕКРЕТНЫЙ!)
# id_ed25519.pub  (публичный ключ - можно делиться)
```

**ИЛИ RSA ключ (если нужна совместимость):**
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

---

## Копирование Ключей на Серверы

**ssh-copy-id (автоматический способ):**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@server.com
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10

# Введет пароль один раз
# Скопирует публичный ключ в ~/.ssh/authorized_keys
```

**Ручной способ:**
```bash
cat ~/.ssh/id_ed25519.pub | ssh ubuntu@server.com \
  "cat >> ~/.ssh/authorized_keys"
```

**Проверка подключения:**
```bash
ssh -i ~/.ssh/id_ed25519 ubuntu@server.com "echo OK"
# Должно вывести: OK

# Подробная отладка
ssh -vvv -i ~/.ssh/id_ed25519 ubuntu@server.com
```

---

## ansible.cfg

**Создать в рабочей директории проекта:**
```ini
[defaults]
# Путь к инвентарю
inventory = ./hosts.ini

# Не проверять SSH ключи (опционально, небезопасно!)
host_key_checking = False

# SSH параметры
remote_user = ubuntu
private_key_file = ~/.ssh/id_ed25519

# Параллелизм
forks = 5

# Timeout
timeout = 30

# Сохранять логи
log_path = ./ansible.log
```

**Проверка конфигурации:**
```bash
ansible-config dump | grep -E "inventory|private_key"
```

---

## SSH Agent (для защищённых ключей)

Если ваш ключ защищен паролем (фраза):

```bash
# Запустить SSH Agent
eval $(ssh-agent -s)

# Добавить ключ (введет пароль один раз)
ssh-add ~/.ssh/id_ed25519

# Проверить добавленные ключи
ssh-add -l

# Использовать в Ansible
ansible -i hosts.ini all -m ping
# Больше не будет просить пароль
```

---

## Дополнительные Коллекции

```bash
# Основные коллекции
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker

# Просмотр установленных
ansible-galaxy collection list

# Обновить коллекцию
ansible-galaxy collection install ansible.posix --upgrade
```

---

## Проблемы и Решения

**"ansible: command not found"**
```bash
# Переустановить через pip
pip install --force-reinstall ansible

# Или добавить в PATH (если нужно)
export PATH=$PATH:~/.local/bin
```

**"Python 3.9 not found" на целевом сервере**
```bash
# На сервере установить Python
ssh ubuntu@server "sudo apt install python3.12"

# Или указать в инвентаре
[webservers]
server1 ansible_python_interpreter=/usr/bin/python3.12
```

**"Permission denied (publickey)"**
```bash
# Проверить что ключ скопирован
ssh ubuntu@server "cat ~/.ssh/authorized_keys | grep $(cat ~/.ssh/id_ed25519.pub)"

# Проверить права на файл
ssh ubuntu@server "ls -la ~/.ssh/"
# Должно быть: drwx------ .ssh (права 700)
#             -rw-r--r-- authorized_keys (права 644)

# Если не так, исправить
ssh ubuntu@server "chmod 700 ~/.ssh && chmod 644 ~/.ssh/authorized_keys"
```

---

**Следующее:** [[03-inventory|Инвентарь и Хосты]]
