---
title: "5 Создание мульти-машинного кластера"
description: "Как описать и запустить несколько ВМ в одном Vagrantfile."
---

Vagrant позволяет поднять целый стенд (например, 1 Master и 2 Workers) одной командой.

## Конфигурация через цикл

Если машины одинаковые, используйте цикл Ruby.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  # Создаем 3 машины: node1, node2, node3
  (1..3).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      # Присваиваем IP: 192.168.56.11, .12, .13
      node.vm.network "private_network", ip: "192.168.56.1#{i}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
      end
    end
  end
end
```

## Конфигурация разных ролей

Если машины разные (например, DB и Web).

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  config.vm.define "db" do |db|
    db.vm.network "private_network", ip: "192.168.56.10"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
    end
  end

  config.vm.define "web" do |web|
    web.vm.network "private_network", ip: "192.168.56.20"
    web.vm.provision "shell", inline: "echo 'Connect to DB at 192.168.56.10'"
  end
end
```

## Управление кластером

Команды теперь принимают имя машины:

*   `vagrant up` — запустить все.
*   `vagrant up db` — запустить только базу данных.
*   `vagrant ssh web` — подключиться к веб-серверу.
