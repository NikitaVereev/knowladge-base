---
title: "Интеграция с Ansible"
type: how-to
tags: [vagrant, ansible, provisioner, inventory, playbook]
sources:
  original: "devops/vagrant/how-to/7-ansible-integration.md"
related:
  - "[[vagrant/how-to/provisioning]]"
  - "[[ansible/index]]"
---

# Интеграция Vagrant с Ansible

> **TL;DR:** Vagrant автоматически генерирует inventory и запускает плейбук.
> Блок `config.vm.provision "ansible"` в Vagrantfile. Или запуск вручную через `ssh-config`.

## Встроенный провижинер

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"
  config.vm.network "private_network", ip: "192.168.56.10"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"

    # Группы хостов
    ansible.groups = {
      "webservers" => ["default"],
      "all:vars"   => { "env" => "dev" }
    }

    # Переменные
    ansible.extra_vars = { http_port: 8080 }

    # Verbose
    ansible.verbose = "v"
  end
end
```

Vagrant автоматически запускает `ansible-playbook` при `vagrant up`.

## Запуск Ansible вручную

```bash
# Сохранить SSH-конфиг
vagrant ssh-config > vagrant.ssh.conf

# Запустить плейбук
ansible-playbook -i "192.168.56.10," playbook.yml \
  --user vagrant \
  --ssh-common-args="-F vagrant.ssh.conf"
```

## Мульти-машинный кластер + Ansible

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  config.vm.define "web" do |web|
    web.vm.network "private_network", ip: "192.168.56.10"
  end

  config.vm.define "db" do |db|
    db.vm.network "private_network", ip: "192.168.56.11"

    # Ansible запускается ТОЛЬКО после последней машины
    db.vm.provision "ansible" do |ansible|
      ansible.playbook = "site.yml"
      ansible.limit = "all"
      ansible.groups = {
        "webservers" => ["web"],
        "databases"  => ["db"]
      }
    end
  end
end
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Ansible не установлен на хосте | `pip install ansible` — Ansible нужен на хосте, не в VM |
| `ansible_local` vs `ansible` | `ansible` = запуск с хоста. `ansible_local` = установка и запуск внутри VM |
| Provisioning не запускается повторно | `vagrant provision` или `vagrant up --provision` |
