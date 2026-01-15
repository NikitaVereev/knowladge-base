---
title: 02 Установка и Конфигурация
---

---

## Установка VirtualBox

**macOS:**
```bash
# Через Homebrew
brew install --cask virtualbox

# Проверить версию
VBoxManage --version
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack

# Проверить
VBoxManage --version
```

**Arch Linux:**
```bash
sudo pacman -S virtualbox virtualbox-host-modules-arch

# Загрузить модули
sudo modprobe vboxdrv

# Проверить
VBoxManage --version
```

**Windows:**
```powershell
# Через Chocolatey
choco install virtualbox

# Или скачать с https://www.virtualbox.org/
```

---

## Установка Vagrant

**macOS:**
```bash
# Через Homebrew (рекомендуется)
brew install vagrant

# Проверить
vagrant version
```

**Ubuntu/Debian:**
```bash
# Добавить официальный репозиторий
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

# Установить
sudo apt-get update
sudo apt-get install vagrant

# Проверить
vagrant version
```

**Arch Linux:**
```bash
# Или из AUR (свежая версия)
yay -S vagrant

# Проверить
vagrant version
```

**Windows:**
```powershell
# Через Chocolatey
choco install vagrant

# Или скачать с https://www.vagrantup.com/downloads
# и установить .msi файл

# Проверить
vagrant version
```

---

## Первая Машина

**1. Инициализировать проект:**
```bash
# Создать директорию
mkdir my-vagrant-project
cd my-vagrant-project

# Инициализировать с Ubuntu 22.04
vagrant init ubuntu/jammy64

# Создаёт Vagrantfile
ls -la Vagrantfile
```

**2. Запустить машину:**
```bash
vagrant up

# Ждёт 10-15 минут
# Скачивает box (если нужен) ~1GB
# Создаёт и запускает ВМ
```

**3. Подключиться по SSH:**
```bash
vagrant ssh

# Вы внутри машины!
$ whoami
vagrant

$ lsb_release -a
Ubuntu 22.04 LTS

$ exit  # Выйти из SSH
```

---

## Базовый Vagrantfile

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  (1..2).each do |i|
    config.vm.define "server#{i}" do |web|
      web.vm.box = "ubuntu/jammy64"
      web.vm.network "forwarded_port", id: "ssh", host: 2222 + i, guest: 22
      web.vm.network "private_network", ip: "10.11.10.#{i}", virtualbox__intnet: true
      web.vm.hostname = "server#{i}"

      web.vm.provision "shell" do |s|
        ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ed25519.pub").first.strip
        s.inline = <<-SHELL
        echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
        echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
        SHELL
      end

      web.vm.provider "virtualbox" do |v|
        v.name = "server#{i}"
        v.memory = 2048
        v.cpus = 1
      end
    end
  end
end
```

---

## Команды Vagrant

| Команда | Описание |
|---------|---------|
| `vagrant init <box>` | Инициализировать новый проект |
| `vagrant up` | Создать и запустить машины |
| `vagrant up server1` | Запустить конкретную машину |
| `vagrant status` | Статус машин |
| `vagrant ssh` | Подключиться по SSH |
| `vagrant ssh server1` | SSH в конкретную машину
| `vagrant halt` | Остановить машины |
| `vagrant destroy` | Удалить машины |
| `vagrant reload` | Перезагрузить машины |
| `vagrant provision` | Выполнить provisioning |
| `vagrant box list` | Список скачанных boxes |
| `vagrant box add <box>` | Скачать box |

---

## Port Forwarding

**Зачем нужен:**
- Доступ к сервисам в ВМ с хоста
- Guest порт = порт в ВМ
- Host порт = порт на твоём ПК

**Пример:**
```ruby
# Веб-сервер в ВМ на 8080
# Доступен на хосте через 8080
config.vm.network "forwarded_port", guest: 8080, host: 8080

# Node.js в ВМ на 3000
# Доступен на хосте через 3000
config.vm.network "forwarded_port", guest: 3000, host: 3000
```

**Использование:**
```bash
# Внутри ВМ
vagrant ssh
python3 -m http.server 8080

# На хосте (другой терминал)
curl http://localhost:8080
# OK!
```

---

## SSH Конфигурация

**Автоматическое подключение:**
```bash
# Vagrant генерирует SSH ключи
vagrant ssh

# Или через стандартный SSH
ssh -i .vagrant/machines/default/virtualbox/private_key \
    -u vagrant 192.168.56.10
```

**Без пароля:**
- Vagrant создаёт пару ключей
- Публичный ключ добавляется в ~/.ssh/authorized_keys в ВМ
- Частный ключ хранится локально

---

## Проверка Установки

```bash
# Версии
vagrant version
VBoxManage --version

# Скачать box
vagrant box add ubuntu/jammy64

# Создать тестовую ВМ
vagrant init ubuntu/jammy64

# Запустить
vagrant up

# Проверить SSH
vagrant ssh -c "echo OK"

# Остановить и удалить
vagrant halt
vagrant destroy
```

---

## Проблемы и Решения

**VirtualBox не установлен**
```bash
# На Ubuntu
sudo apt install virtualbox

# На macOS
brew install --cask virtualbox

# Нужны права администратора!
```

**Vagrant не находит VirtualBox**
```bash
# Обычно нужна перезагрузка после установки
sudo reboot

# Или проверить PATH
which vagrant
which VBoxManage
```

**Ошибка "vagrant: command not found"**
```bash
# Переустановить через пакетный менеджер
# macOS:
brew install vagrant

# Ubuntu:
sudo apt install vagrant
```

**Slow port forwarding**
```ruby
# Увеличить timeout
config.vm.network "forwarded_port", guest: 8080, host: 8080,
  auto_correct: true
```

---

**Следующее:** [[03-machine-lifecycle|Жизненный цикл Машины]]
