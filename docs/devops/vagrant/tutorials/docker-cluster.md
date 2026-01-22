---
title: "1 Развертывание Docker-кластера"
description: "Практическое руководство по созданию 3-х виртуальных машин с предустановленным Docker."
---

В этом уроке вы создадите полностью готовый к работе кластер из трех серверов Ubuntu, на каждом из которых будет установлен Docker Engine.

**Чему вы научитесь:**
*   Использовать циклы в Vagrantfile.
*   Использовать Shell-провижининг для установки ПО.
*   Настраивать частную сеть между машинами.

## Шаг 1: Подготовка скрипта установки

Мы не будем устанавливать Docker вручную. Вместо этого мы опишем установку в переменной Ruby прямо в `Vagrantfile`.

## Шаг 2: Создание Vagrantfile

Создайте файл `Vagrantfile` и вставьте следующий код:

```ruby
Vagrant.configure("2") do |config|
  # 1. Скрипт установки Docker (Official way)
  # Удаляем конфликты, ставим ключи, репозиторий и сам Docker
  $install_docker = <<-SHELL
    # Удаляем старые версии, если есть
    for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove $pkg; done

    # Ставим зависимости
    apt-get update
    apt-get install -y ca-certificates curl gnupg

    # Добавляем GPG ключ Docker
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg

    # Добавляем репозиторий
    echo \
      "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
      tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Устанавливаем Docker Engine
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

    # Добавляем пользователя vagrant в группу docker (чтобы работать без sudo)
    usermod -aG docker vagrant
  SHELL

  # 2. Описание кластера
  (1..3).each do |i|
    config.vm.define "docker-#{i}" do |node|
      node.vm.box = "ubuntu/jammy64"
      node.vm.hostname = "docker-#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{20 + i}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 2
      end
      
      # Запуск установки Docker
      node.vm.provision "shell", inline: $install_docker

      # 3. Проброс вашего SSH-ключа
      # Читаем ключ с хост-машины и кладем в authorized_keys внутри ВМ
      node.vm.provision "shell" do |s|
        # Укажите путь к вашему публичному ключу
        ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ed25519.pub").first.strip
        s.inline = <<-SHELL
          echo "#{ssh_pub_key}" >> /home/vagrant/.ssh/authorized_keys
          # Опционально: добавить для root, если требуется
          # mkdir -p /root/.ssh
          # echo "#{ssh_pub_key}" >> /root/.ssh/authorized_keys
        SHELL
      end
    end
  end
end
```

## Шаг 3: Запуск кластера

Выполните команду:
```bash
vagrant up
```
Vagrant скачает образ Ubuntu, запустит три машины по очереди и на каждой выполнит скрипт установки Docker. Это займет 5-10 минут.

## Шаг 4: Проверка результата

1.  Подключитесь к первой ноде:
    ```bash
    vagrant ssh docker-1
    ```
2.  Проверьте, работает ли Docker:
    ```bash
    docker run hello-world
    ```
3.  Проверьте связь с соседями:
    ```bash
    ping -c 3 192.168.56.22  # пинг до docker-2
    ```

**Поздравляем!** У вас есть локальный кластер для экспериментов с Docker Swarm или Kubernetes.
