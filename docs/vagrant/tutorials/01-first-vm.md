---
title: "01 — Первая виртуальная машина"
type: tutorial
tags: [vagrant, tutorial, virtualbox, install, vm, first-steps]
sources:
  original: "devops/virtualbox/how-to/create-vm-manually.md + devops/vagrant/how-to/1-install.md + 2-basic-workflow.md"
related:
  - "[[vagrant/explanation/what-is-vagrant]]"
  - "[[vagrant/tutorials/02-docker-cluster]]"
  - "[[vagrant/how-to/configure-vagrantfile]]"
---

# Tutorial 01 — Первая виртуальная машина

> **Цель:** Установить VirtualBox + Vagrant. Поднять Ubuntu VM одной командой.
> Подключиться по SSH, поработать, удалить.

**Время:** ~20 минут

## Шаг 1. Установка VirtualBox

### Arch Linux

```bash
sudo pacman -Syu
sudo pacman -S linux-headers virtualbox virtualbox-host-modules-arch
sudo modprobe vboxdrv
sudo usermod -aG vboxusers $USER
# Перелогиньтесь для применения группы
```

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack
```

### macOS

```bash
brew install --cask virtualbox
# Разрешить в System Settings → Privacy & Security если потребуется
```

### Проверка

```bash
VBoxManage --version
```

## Шаг 2. Установка Vagrant

### macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vagrant
```

### Ubuntu / Debian

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

### Arch Linux

```bash
yay -S vagrant
```

### Проверка

```bash
vagrant --version
# Vagrant 2.4.x
```

## Шаг 3. Первая VM

```bash
mkdir ~/vagrant-lab && cd ~/vagrant-lab

# Создать Vagrantfile (Ubuntu 24.04)
vagrant init ubuntu/noble64

# Запустить (скачает образ при первом запуске)
vagrant up
```

## Шаг 4. Подключение

```bash
# SSH в VM
vagrant ssh

# Внутри VM:
whoami          # vagrant
uname -a        # Ubuntu kernel
ip a            # сетевые интерфейсы
exit            # выйти
```

## Шаг 5. Жизненный цикл

```bash
# Приостановить (сохранить RAM на диск)
vagrant suspend

# Возобновить
vagrant resume

# Выключить (как shutdown)
vagrant halt

# Запустить снова
vagrant up

# Статус
vagrant status

# Уничтожить (удалить VM, Vagrantfile остаётся)
vagrant destroy -f

# Пересоздать с нуля
vagrant up
```

## Шаг 6. Подключение внешним SSH-клиентом

```bash
vagrant ssh-config
# HostName 127.0.0.1
# User vagrant
# Port 2222
# IdentityFile /path/to/.vagrant/machines/default/virtualbox/private_key

# Подключиться напрямую
ssh -p 2222 -i .vagrant/machines/default/virtualbox/private_key vagrant@127.0.0.1
```

## Что мы изучили

| Концепция | Что сделали |
|-----------|------------|
| VirtualBox | Гипервизор, запускает VM |
| Vagrant | Оркестратор, управляет VirtualBox через CLI |
| Box | Предсобранный образ (`ubuntu/noble64`) |
| Vagrantfile | Конфигурация VM (создаётся `vagrant init`) |
| Жизненный цикл | up → ssh → halt/suspend → destroy |

## Что дальше

→ [[vagrant/how-to/configure-vagrantfile]] — CPU, RAM, сеть, synced folders
→ [[vagrant/tutorials/02-docker-cluster]] — кластер из 3 VM с Docker

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `VT-x is not available` | Включить виртуализацию в BIOS (VT-x / AMD-V) |
| `vboxdrv` не загружается (Arch) | `sudo pacman -S linux-headers && sudo modprobe vboxdrv` |
| Box долго скачивается | Нормально при первом запуске (~500MB-1GB) |
| `vagrant up` зависает | Проверить VirtualBox: `VBoxManage list runningvms` |