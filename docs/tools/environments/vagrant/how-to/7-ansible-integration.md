---
title: "7 Интеграция с Ansible"
description: "Как запускать Ansible Playbooks на машинах Vagrant."
---

Vagrant имеет встроенный провижинер для Ansible, который сам генерирует inventory-файл и запускает плейбук.

## Настройка Vagrantfile

Добавьте этот блок в конфигурацию вашей машины:

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
  
  # Создание групп хостов для Ansible
  ansible.groups = {
    "webservers" => ["my-vm"],
    "all_vars" => ["webservers"]
  }
  
  # Передача переменных
  ansible.extra_vars = {
    http_port: 8080
  }
end
```

Vagrant автоматически запустит `ansible-playbook` на хост-машине при `vagrant up`.

## Запуск Ansible вручную

Если вы хотите запускать Ansible отдельно, вам нужен корректный inventory. Проще всего сгенерировать `ssh-config`:

1.  Сохраните конфиг:
    ```bash
    vagrant ssh-config > vagrant.ssh.conf
    ```
2.  Запустите Ansible, используя этот конфиг:
    ```bash
    ansible-playbook -i 127.0.0.1, playbook.yml \
      --user vagrant \
      --ssh-common-args="-F vagrant.ssh.conf"
    ```
