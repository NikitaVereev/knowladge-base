---
title: 04 Создание Кластера Серверов
---

---

## Многомашинная Конфигурация

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  
  # Базовые настройки для всех машин
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  # Цикл создания 5 машин (1-5)
  (1..5).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}.local"
      
      # IP адреса: 10, 11, 12, 13, 14
      node.vm.network "private_network", 
        ip: "192.168.56.#{10 + i - 1}"
      
      # Port forwarding для каждой машины
      node.vm.network "forwarded_port", guest: 22, host: "222#{i}"
      node.vm.network "forwarded_port", guest: 8080, host: "808#{i}"
    end
  end
end
```

---

## Запуск Кластера

```bash
# Запустить все машины
vagrant up

# Запустить конкретные машины
vagrant up node1 node2

# Проверить статус всех
vagrant status

# Подключиться к node3
vagrant ssh node3

# Остановить всех
vagrant halt

# Удалить всех
vagrant destroy
```

---

## Требования Оборудования

| Машин | RAM | Диск | CPU | Время |
|--------|-----|------|-----|-------|
| 1 | 3GB | 25GB | 2 | 10 min |
| 2 | 5GB | 50GB | 4 | 15 min |
| 3 | 7GB | 75GB | 4 | 20 min |
| 5 | 11GB | 125GB | 8 | 30 min |

**Рекомендации:**
- **Минимум:** 8GB RAM + 128GB диск
- **Оптимально:** 16GB RAM + 256GB SSD
- **Полный:** 32GB RAM + 512GB+ SSD

---

## Интеграция с Ansible

**Генерация инвентаря автоматически:**
```bash
# Vagrant может экспортировать инвентарь
vagrant ssh-config > ssh_config

# Или создать вручную hosts.ini:
[cluster]
node1 ansible_host=192.168.56.11 ansible_user=vagrant
node2 ansible_host=192.168.56.12 ansible_user=vagrant
node3 ansible_host=192.168.56.13 ansible_user=vagrant
node4 ansible_host=192.168.56.14 ansible_user=vagrant
node5 ansible_host=192.168.56.15 ansible_user=vagrant
```

**Проверка доступности:**
```bash
ansible all -i hosts.ini -m ping
```

**Запуск playbook'а:**
```bash
ansible-playbook -i hosts.ini playbook.yml
```

---

## Provisioning с Ansible

```ruby
Vagrant.configure("2") do |config|
  # ... определение машин ...
  
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.inventory_path = "hosts.ini"
    ansible.host_key_checking = false
    
    # Для каждого хоста
    ansible.groups = {
      "all" => ["node1", "node2", "node3", "node4", "node5"],
      "managers" => ["node1"],
      "workers" => ["node2", "node3", "node4", "node5"]
    }
  end
end
```

---

## Пример: Docker на Кластере

**Vagrantfile:**
```ruby
Vagrant.configure("2") do |config|
  (1..3).each do |i|
    config.vm.define "docker#{i}" do |node|
      node.vm.box = "ubuntu/jammy64"
      node.vm.hostname = "docker#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{20 + i}"
      
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 2
      end
      
      # Provisioning Docker
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl software-properties-common
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
        apt-get update
        apt-get install -y docker-ce docker-ce-cli containerd.io
        usermod -aG docker vagrant
      SHELL
    end
  end
end
```

**Запуск:**
```bash
vagrant up

# Проверить Docker
vagrant ssh docker1 -c "docker --version"

# На всех машинах
for i in 1 2 3; do
  echo "docker$i:"
  vagrant ssh docker$i -c "docker ps"
done
```

---

## Сетевая Конфигурация

**Private Network (для машин между собой):**
```ruby
config.vm.network "private_network", ip: "192.168.56.10"
```

**Port Forwarding (доступ с хоста):**
```ruby
config.vm.network "forwarded_port", guest: 8080, host: 8080
```

**Public Network (доступ из сети):**
```ruby
config.vm.network "public_network", bridge: "en0"
```

---

## Оптимизация Скорости

**Отключить gather facts в Ansible:**
```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "site.yml"
  ansible.raw_arguments = ["--fact-path=/tmp/ansible/facts"]
end
```

**Использовать rsync для файлов:**
```ruby
config.vm.synced_folder ".", "/vagrant", type: "rsync",
  rsync__exclude: [".git/", ".vagrant/"]
```

**Параллельный запуск:**
```bash
VAGRANT_PARALLEL=true vagrant up
```

---

**Следующее:** [[05-docker-swarm|Подготовка Docker Swarm Кластера]]
