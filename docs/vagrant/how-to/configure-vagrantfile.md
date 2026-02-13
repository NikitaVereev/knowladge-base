---
title: "Настройка Vagrantfile"
type: how-to
tags: [vagrant, vagrantfile, config, cpu, ram, network, synced-folders, ssh]
sources:
  original: "devops/vagrant/how-to/3-configure-vagrantfile.md + 6-connect-ssh.md"
related:
  - "[[vagrant/tutorials/01-first-vm]]"
  - "[[vagrant/explanation/networking]]"
  - "[[vagrant/how-to/provisioning]]"
---

# Настройка Vagrantfile

> **TL;DR:** Ресурсы (CPU/RAM) → блок `provider`. Сеть → `config.vm.network`.
> Общие папки → `synced_folder`. После изменений: `vagrant reload`.

## Ресурсы (CPU / RAM)

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "my-dev-vm"    # имя в VirtualBox GUI
    vb.memory = "2048"       # RAM в MB
    vb.cpus = 2              # ядра
    vb.gui = false           # headless (без окна)
  end
end
```

## Сеть

```ruby
# Проброс портов (NAT)
config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
config.vm.network "forwarded_port", guest: 3000, host: 3000

# Статический IP (Private Network) — рекомендуется для кластеров
config.vm.network "private_network", ip: "192.168.56.10"

# Bridged (Public Network) — VM в LAN
config.vm.network "public_network"
```

## Синхронизация папок

```ruby
# Текущая папка хоста → /var/www в VM
config.vm.synced_folder ".", "/var/www"

# Конкретная папка
config.vm.synced_folder "./src", "/app/src"

# Отключить стандартную синхронизацию /vagrant
config.vm.synced_folder ".", "/vagrant", disabled: true
```

## Hostname

```ruby
config.vm.hostname = "dev-server"
```

## SSH подключение из внешнего клиента

```bash
# Получить параметры
vagrant ssh-config
# Host default
#   HostName 127.0.0.1
#   User vagrant
#   Port 2222
#   IdentityFile /path/to/private_key

# Подключиться напрямую
ssh -p 2222 -i .vagrant/machines/default/virtualbox/private_key vagrant@127.0.0.1

# Копирование файлов
scp -P 2222 -i .vagrant/.../private_key file.txt vagrant@127.0.0.1:/home/vagrant/
```

## Полный пример

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"
  config.vm.hostname = "webserver"

  config.vm.network "private_network", ip: "192.168.56.10"
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.synced_folder "./app", "/var/www/app"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y nginx
  SHELL
end
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Изменения не применились | `vagrant reload` (или `vagrant reload --provision`) |
| Порт занят | `auto_correct: true` — Vagrant найдёт свободный |
| Synced folder медленный | NFS: `config.vm.synced_folder ".", "/app", type: "nfs"` |
| `vagrant ssh` не работает | `vagrant ssh-config` → проверить порт и ключ |