---
title: "02 — Docker-кластер на Vagrant"
type: tutorial
tags: [vagrant, tutorial, docker, cluster, multi-machine, provisioning]
sources:
  original: "devops/vagrant/tutorials/docker-cluster.md"
related:
  - "[[vagrant/tutorials/01-first-vm]]"
  - "[[vagrant/how-to/multi-machine]]"
  - "[[vagrant/how-to/provisioning]]"
  - "[[docker/index]]"
---

# Tutorial 02 — Docker-кластер на Vagrant

> **Цель:** Поднять 3 VM с Docker, Private Network, SSH-ключами. Проверить связность.

**Время:** ~25 минут
**Требования:** Пройден Tutorial 01. VirtualBox + Vagrant установлены.

## Шаг 1. Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  # Скрипт установки Docker (Official way)
  $install_docker = <<-SHELL
    for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
      apt-get remove -y $pkg 2>/dev/null
    done

    apt-get update
    apt-get install -y ca-certificates curl gnupg

    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    usermod -aG docker vagrant
  SHELL

  # 3 ноды
  (1..3).each do |i|
    config.vm.define "docker-#{i}" do |node|
      node.vm.box = "ubuntu/jammy64"
      node.vm.hostname = "docker-#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{20 + i}"

      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 2
      end

      node.vm.provision "shell", inline: $install_docker

      # Проброс SSH-ключа хоста
      node.vm.provision "shell" do |s|
        ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ed25519.pub").first.strip
        s.inline = <<-SHELL
          echo "#{ssh_pub_key}" >> /home/vagrant/.ssh/authorized_keys
        SHELL
      end
    end
  end
end
```

## Шаг 2. Запуск

```bash
vagrant up
# 5-10 минут: скачивание образа + установка Docker на каждую ноду
```

## Шаг 3. Проверка

```bash
# Подключиться к первой ноде
vagrant ssh docker-1

# Docker работает?
docker run hello-world

# Связь с соседями?
ping -c 3 192.168.56.22    # docker-2
ping -c 3 192.168.56.23    # docker-3

exit
```

## Шаг 4. Управление кластером

```bash
vagrant status              # статус всех
vagrant ssh docker-2        # подключиться к конкретной
vagrant halt                # выключить все
vagrant up docker-1         # запустить только одну
vagrant destroy -f          # удалить все
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| Multi-machine | Цикл `(1..3).each` для создания нескольких VM |
| Shell provisioning | `$install_docker` — автоустановка Docker |
| Private network | Статические IP `192.168.56.2x`, VM видят друг друга |
| SSH key provisioning | Проброс хостового ключа внутрь VM |

## Что дальше

Этот кластер готов для: Docker Swarm, тестирования Ansible-плейбуков, экспериментов с Kubernetes (k3s).
