---
title: "Provisioning"
type: how-to
tags: [vagrant, provisioning, shell, ansible, script, automation]
sources:
  original: "devops/vagrant/how-to/4-provisioning.md"
related:
  - "[[vagrant/how-to/configure-vagrantfile]]"
  - "[[vagrant/how-to/ansible-integration]]"
---

# Provisioning (автонастройка VM)

> **TL;DR:** Provisioning превращает чистую ОС в готовый сервер. Shell inline, внешний скрипт или Ansible.
> Запускается при первом `vagrant up` или принудительно `vagrant provision`.

## Shell Inline

```ruby
config.vm.provision "shell", inline: <<-SHELL
  apt-get update
  apt-get install -y nginx git vim
  echo "Hello World" > /var/www/html/index.html
SHELL
```

Скрипт выполняется от **root** по умолчанию.

## Внешний скрипт

```ruby
config.vm.provision "shell", path: "bootstrap.sh"
```

```bash
# bootstrap.sh
#!/bin/bash
set -euo pipefail
apt-get update
apt-get install -y docker.io
usermod -aG docker vagrant
```

## От обычного пользователя

```ruby
config.vm.provision "shell", inline: "echo $HOME", privileged: false
```

## Когда запускается

| Команда | Provisioning |
|---------|-------------|
| `vagrant up` (первый раз) | ✅ Да |
| `vagrant up` (повторно) | ❌ Нет |
| `vagrant provision` | ✅ Принудительно |
| `vagrant reload --provision` | ✅ Перезагрузка + provision |
| `vagrant destroy && vagrant up` | ✅ С нуля |

## Несколько провижинеров

```ruby
config.vm.provision "shell", inline: "apt-get update"
config.vm.provision "shell", path: "install-docker.sh"
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end
```

Выполняются последовательно.

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Скрипт не идемпотентный | `apt-get install -y` (без интерактива), проверки `if ! command -v ...` |
| Provisioning не запускается | `vagrant provision` или `vagrant up --provision` |
| `E: Could not get lock` | Подождать — возможно автообновление в фоне (Ubuntu) |
