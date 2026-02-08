---
title: "06. Создание Swarm-кластера"
type: tutorial
tags: [docker, swarm, tutorial, vagrant, cluster, high-availability]
sources:
  docs: "https://docs.docker.com/engine/swarm/swarm-tutorial/"
related:
  - "[[docker/explanation/swarm]]"
  - "[[docker/how-to/swarm/deploy-stack]]"
  - "[[docker/reference/swarm-cheatsheet]]"
  - "[[vagrant/how-to/configure-vagrantfile]]"
prev: "[[docker/tutorials/05-debugging-workshop]]"
next: null
---


В этом уроке мы объединим навыки Vagrant и Docker, чтобы создать тестовую лабораторию Swarm.

## Шаг 1: Vagrantfile

Мы используем конфиг с установкой Docker и добавлением SSH-ключа.

```ruby
Vagrant.configure("2") do |config|
  # Скрипт установки Docker (Official way)
  $install_docker = <<-SHELL
    for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove $pkg; done
    apt-get update
    apt-get install -y ca-certificates curl gnupg
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    usermod -aG docker vagrant
  SHELL

  (1..3).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.box = "ubuntu/jammy64" # Ubuntu 22.04 LTS
      node.vm.hostname = "node-#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{10 + i}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
        vb.cpus = 1
      end
      
      node.vm.provision "shell", inline: $install_docker
    end
  end
end
```

Запустите стенд: `vagrant up`.

## Шаг 2: Инициализация (Node-1)

Зайдите на первую ноду: `vagrant ssh node-1` (или просто `ssh vagrant@192.168.56.11`).

Выполните инициализацию, указав IP адрес частной сети:
```bash
docker swarm init --advertise-addr 192.168.56.11
```

Вы получите команду с токеном. Скопируйте её. Пример:
`docker swarm join --token SWMTKN-1-xxxxx 192.168.56.11:2377`

## Шаг 3: Добавление воркеров (Node-2, Node-3)

Зайдите на остальные ноды и выполните скопированную команду.

```bash
# На node-2

> **Цель:** Поднять 3-нодовый Swarm-кластер (Vagrant/VirtualBox), задеплоить стек,
> увидеть rolling update и self-healing в действии.
docker swarm join --token SWMTKN-1-xxxxx 192.168.56.11:2377

# На node-3
docker swarm join --token SWMTKN-1-xxxxx 192.168.56.11:2377
```

## Шаг 4: Проверка

Вернитесь на **Node-1** и проверьте список узлов:

```bash
docker node ls
```

Вы должны увидеть 3 ноды со статусом `Ready` и `Active`. Поздравляем, ваш кластер готов к деплою стеков!

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `docker swarm join` с неправильным токеном | `Error: invalid join token` | Получить свежий токен: `docker swarm join-token worker` на менеджере |
| Воркер не видит менеджер | `connection refused` при join | Проверить: IP менеджера верный? Порт 2377 открыт? Файрвол? |
| `docker stack deploy` с `build:` | `unsupported: build` | Swarm не собирает образы. Использовать `image:` из registry |
| Потеряли кворум (2 из 3 менеджеров упали) | Кластер read-only, нельзя деплоить | `docker swarm init --force-new-cluster` на оставшемся менеджере |
