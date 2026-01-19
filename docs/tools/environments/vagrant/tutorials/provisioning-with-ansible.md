---
title: "Vagrant c Ansible Provisioning"
description: "Как настроить Vagrantfile для автоматического запуска Ansible Playbook при создании виртуальной машины."
---


Vagrant умеет не только создавать виртуальные машины, но и сразу "накатывать" на них конфигурацию через Ansible. Это позволяет получать готовое рабочее окружение одной командой `vagrant up`.

## Структура проекта

```text
project/
  Vagrantfile
  ansible/
    playbooks/
      install-docker.yml
    inventory/
      hosts  (не обязателен, Vagrant генерирует свой)
```

## Настройка Vagrantfile

В блоке конфигурации добавьте секцию `ansible`.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64" # Ubuntu 24.04
  config.vm.network "private_network", ip: "192.168.56.10"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  # Provisioning через Ansible
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/playbooks/install-docker.yml"
    # Vagrant сам создаст инвентарь для VM, но можно указать свой
    # ansible.inventory_path = "ansible/inventory/hosts"
    ansible.verbose = "v"
  end
end
```

## Запуск

```bash
vagrant up
```

Если машина уже запущена и вы хотите только применить изменения в Ansible:

```bash
vagrant provision
```

## Особенности

1. **Автоматический инвентарь:** Vagrant сам генерирует inventory-файл в `.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory`, где прописан хост, SSH-ключи и порты. Вам не нужно настраивать SSH-доступ вручную.
2. **Пользователь:** Playbook будет выполняться от пользователя `vagrant`, поэтому задачи, требующие root, должны иметь `become: yes`.

## Связанные материалы

- [[tools/environments/vagrant/tutorials/getting-started|Основы Vagrant]]
- [[devops/ansible/tutorials/install-docker-playbook|Playbook установки Docker]]
