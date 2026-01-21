---
title: "4 Автоматическая настройка (Provisioning)"
description: "Как запускать скрипты настройки при старте машины."
---

Provisioning позволяет превратить "чистую" ОС в готовый сервер автоматически.

## Использование Shell-скриптов

Самый простой способ — написать команды Bash прямо в `Vagrantfile`.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"

  # Скрипт выполняется от root
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y nginx
    echo "Hello World" > /var/www/html/index.html
  SHELL
end
```

## Внешний скрипт

Для больших скриптов лучше вынести код в отдельный файл (например, `bootstrap.sh` рядом с Vagrantfile).

```ruby
config.vm.provision "shell", path: "bootstrap.sh"
```

## Когда запускается провижининг?

1.  При **первом** запуске `vagrant up`.
2.  Принудительно командой `vagrant provision`.
3.  При перезагрузке с флагом `vagrant reload --provision`.

*Совет: Пишите скрипты идемпотентными (чтобы повторный запуск не ломал систему).*
