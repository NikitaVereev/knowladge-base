---
title: "3 Настройка Vagrantfile"
description: "Рецепты конфигурации: ресурсы (CPU/RAM), сеть и порты."
---

Все настройки производятся в файле `Vagrantfile`. После изменения настроек всегда выполняйте `vagrant reload`, чтобы применить их.

## Настройка ресурсов (CPU/RAM)

Настройки зависят от провайдера. Для VirtualBox:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  config.vm.provider "virtualbox" do |vb|
    vb.name = "my-dev-vm" # Имя в GUI VirtualBox
    vb.memory = "2048"    # Память в МБ
    vb.cpus = 2           # Количество ядер
    vb.gui = false        # Запуск без окна (headless)
  end
end
```

## Настройка сети

### Проброс портов
Доступ к 80 порту ВМ через 8080 на хосте:
```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
```

### Статический IP (Private Network)
```ruby
config.vm.network "private_network", ip: "192.168.56.10"
```

### Доступ к LAN (Public Network)
```ruby
config.vm.network "public_network"
```

## Синхронизация папок

Монтирование папки проекта в `/var/www` внутри ВМ:
```ruby
# "." - текущая папка хоста, "/var/www" - путь в ВМ
config.vm.synced_folder ".", "/var/www"
```
