---
title: "Мульти-машинный кластер"
type: how-to
tags: [vagrant, multi-machine, cluster, loop, roles]
sources:
  original: "devops/vagrant/how-to/5-create-cluster.md"
related:
  - "[[vagrant/tutorials/02-docker-cluster]]"
  - "[[vagrant/how-to/configure-vagrantfile]]"
  - "[[vagrant/how-to/ansible-integration]]"
---

# Мульти-машинный кластер

> **TL;DR:** Цикл `(1..N).each` для одинаковых нод. Блоки `config.vm.define` для разных ролей.
> Управление: `vagrant up node1`, `vagrant ssh node2`.

## Одинаковые ноды (цикл)

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  (1..3).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "192.168.56.1#{i}"

      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
        vb.cpus = 1
      end
    end
  end
end
```

## Разные роли

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  config.vm.define "db" do |db|
    db.vm.hostname = "db"
    db.vm.network "private_network", ip: "192.168.56.10"
    db.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
    end
    db.vm.provision "shell", inline: "apt-get update && apt-get install -y postgresql"
  end

  config.vm.define "web" do |web|
    web.vm.hostname = "web"
    web.vm.network "private_network", ip: "192.168.56.20"
    web.vm.network "forwarded_port", guest: 80, host: 8080
    web.vm.provision "shell", inline: "apt-get update && apt-get install -y nginx"
  end
end
```

## Управление

```bash
vagrant up                  # запустить все
vagrant up db               # только БД
vagrant ssh web             # подключиться к web
vagrant halt                # выключить все
vagrant destroy -f          # удалить все
vagrant status              # статус каждой VM
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| IP-конфликт между нодами | Каждой ноде уникальный IP в одной подсети |
| Не хватает RAM на хосте | Уменьшить `vb.memory` или поднимать не все ноды |
| Provisioning одной ноды зависит от другой | Ansible лучше подходит для оркестрации зависимостей |