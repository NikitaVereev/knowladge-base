---
title: 05 Подготовка Docker Swarm Кластера
---

---

## Docker Swarm Архитектура

```
                    Docker Swarm
                        │
        ┌───────────────┼───────────────┐
        │               │               │
     Manager         Worker1         Worker2
     (node1)         (node2)         (node3)
        │               │               │
    Orch.           Services        Services
    Scheduler         Running         Running
    Manager DB          │               │
        │               │               │
     Tasks           Tasks           Tasks
```

---

## Инициализация Swarm

**На Manager ноде (node1):**
```bash
vagrant ssh node1

# Инициализировать Swarm
docker swarm init --advertise-addr 192.168.56.11

# Output:
# Swarm initialized: current node (xxx) is now a manager.
# Token: SWMTKN-1-xxx

# Получить token для worker'ов
docker swarm join-token worker

# Output:
# To add a worker to this swarm, run the following command:
# docker swarm join --token SWMTKN-1-xxx 192.168.56.11:2377
```

**На Worker нодах (node2, node3, node4, node5):**
```bash
vagrant ssh node2

# Присоединиться к Swarm
docker swarm join --token SWMTKN-1-xxx 192.168.56.11:2377

# Output:
# This node joined a swarm as a worker.

# Повторить для node3, node4, node5
```

---

## Проверка Swarm

**На Manager ноде:**
```bash
docker node ls

# Output:
# ID              HOSTNAME   STATUS   AVAILABILITY   MANAGER STATUS
# abc123*         node1      Ready    Active         Leader
# def456          node2      Ready    Active         
# ghi789          node3      Ready    Active
# jkl012          node4      Ready    Active
# mno345          node5      Ready    Active
```

---

## Полный Vagrantfile для Docker Swarm

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  
  (1..5).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}.local"
      node.vm.network "private_network", ip: "192.168.56.#{10 + i - 1}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 2
      end
      
      # Установка Docker
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl software-properties-common
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        apt-get update
        apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
        usermod -aG docker vagrant
        systemctl start docker
        systemctl enable docker
      SHELL
      
      # Инициализировать Swarm на первой ноде
      if i == 1
        node.vm.provision "shell", inline: <<-SHELL
          docker swarm init --advertise-addr 192.168.56.11
        SHELL
      else
        # Присоединить к Swarm (нужен актуальный token!)
        node.vm.provision "shell", inline: <<-SHELL
          # Token можно получить вручную или через скрипт
          # Пока пропустим, выполним вручную
        SHELL
      end
    end
  end
end
```

---

## Ansible Playbook для Swarm

**swarm-init.yml:**
```yaml
---
- name: Initialize Docker Swarm
  hosts: all
  become: yes
  
  tasks:
    - name: Initialize Swarm on manager
      shell: docker swarm init --advertise-addr {{ manager_ip }}
      when: inventory_hostname == groups['managers'][0]
      register: swarm_init
    
    - name: Get worker token
      shell: docker swarm join-token -q worker
      when: inventory_hostname == groups['managers'][0]
      register: worker_token
    
    - name: Join workers to Swarm
      shell: "docker swarm join --token {{ hostvars[groups['managers'][0]]['worker_token'].stdout }} {{ manager_ip }}:2377"
      when: inventory_hostname != groups['managers'][0]
```

**hosts.ini:**
```ini
[managers]
node1 ansible_host=192.168.56.11

[workers]
node2 ansible_host=192.168.56.12
node3 ansible_host=192.168.56.13
node4 ansible_host=192.168.56.14
node5 ansible_host=192.168.56.15

[cluster:children]
managers
workers

[cluster:vars]
manager_ip=192.168.56.11
ansible_user=vagrant
ansible_ssh_private_key_file=.vagrant/machines/*/virtualbox/private_key
```

---

## Запуск Сервиса в Swarm

**Создать сервис:**
```bash
# На manager (node1)
vagrant ssh node1

# Создать простой сервис
docker service create --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx

# Проверить сервис
docker service ls
docker service ps web
```

**Масштабирование:**
```bash
# Увеличить количество реплик
docker service scale web=5

# Проверить распределение
docker service ps web
```

---

## Развёртывание Приложения

**docker-compose.yml для Swarm:**
```yaml
version: '3.9'

services:
  web:
    image: nginx
    ports:
      - "8080:80"
    deploy:
      replicas: 3
      placement:
        constraints: [node.role == worker]
  
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
```

**Развернуть:**
```bash
docker stack deploy -c docker-compose.yml myapp

# Проверить
docker stack ls
docker stack services myapp
docker stack ps myapp
```

---

## Мониторинг Swarm

```bash
# Список нод
docker node ls

# Информация о ноде
docker node inspect node1

# Список сервисов
docker service ls

# Задачи в сервисе
docker service ps web

# Логи сервиса
docker service logs web

# Статистика
docker stats
```

---

## Практический Workflow

```bash
# 1. Запустить кластер
vagrant up

# 2. Подождать инициализацию (5-10 мин)
vagrant status

# 3. На node1: инициализировать Swarm
vagrant ssh node1
docker swarm init --advertise-addr 192.168.56.11
docker swarm join-token worker
exit

# 4. На node2-node5: присоединиться
for i in 2 3 4 5; do
  vagrant ssh node$i
  docker swarm join --token SWMTKN-... 192.168.56.11:2377
  exit
done

# 5. Проверить Swarm
vagrant ssh node1
docker node ls

# 6. Создать сервис
docker service create --name web --replicas 3 -p 8080:80 nginx
docker service ps web

# 7. Проверить с хоста
curl http://localhost:8080

# 8. Масштабировать
docker service scale web=5
docker service ps web

# 9. Удалить кластер когда готово
vagrant destroy
```

---

## Troubleshooting

**Swarm join failed:**
```bash
# Проверить token
docker swarm join-token worker

# Проверить IP и порт доступны
curl 192.168.56.11:2377

# Перезагрузить ноду
vagrant reload node2
```

**Сервис не распределяется:**
```bash
# Проверить constraints
docker service inspect web

# Проверить доступность нод
docker node ls

# Если нода down, вернуть её
docker node update --availability active node2
```

---