---
title: 03 Жизненный цикл Машины
---

---

## Основные Команды

```
┌──────────────────────────────────────┐
│      ЖИЗНЕННЫЙ ЦИКЛ VM              │
├──────────────────────────────────────┤
│  init      → Инициализация           │
│  up        → Создание и запуск       │
│  status    → Проверка статуса        │
│  ssh       → Подключение             │
│  provision → Переподготовка          │
│  reload    → Перезагрузка            │
│  halt      → Остановка               │
│  destroy   → Удаление                │
└──────────────────────────────────────┘
```

---

## Детальный Workflow

**1. Инициализация:**
```bash
vagrant init ubuntu/jammy64
# Создаёт Vagrantfile в текущей директории
```

**2. Запуск:**
```bash
vagrant up
# Скачивает box если нужен
# Создаёт ВМ в VirtualBox
# Выполняет provisioning (если указан)
# Машина готова к работе
```

**3. Проверка статуса:**
```bash
vagrant status
# Output:
# Current machine states:
# default  running (virtualbox)
```

**4. SSH подключение:**
```bash
vagrant ssh
# Внутри машины
$ whoami
vagrant

$ pwd
/home/vagrant

$ exit
```

**5. Работа с машиной:**
```bash
# Выполнить команду без SSH
vagrant ssh -c "uptime"

# Скопировать файл
vagrant ssh -c "cat /etc/hostname"

# Повторить provisioning
vagrant provision
```

**6. Перезагрузка:**
```bash
# Перезагрузить ВМ
vagrant reload

# Перезагрузить и запустить provisioning
vagrant reload --provision
```

**7. Остановка:**
```bash
# Graceful shut down
vagrant halt

# Force shut down
vagrant halt -f

# Машина сохранена, занимает место на диске
```

**8. Удаление:**
```bash
# Удалить машину (не удаляет box)
vagrant destroy

# Без подтверждения
vagrant destroy -f

# Освобождается место на диске
```

---

## Многомашинная Конфигурация

```ruby
Vagrant.configure("2") do |config|
  # Машина 1
  config.vm.define "web1" do |web1|
    web1.vm.box = "ubuntu/jammy64"
    web1.vm.hostname = "web1.local"
    web1.vm.network "private_network", ip: "192.168.56.10"
  end
  
  # Машина 2
  config.vm.define "db1" do |db1|
    db1.vm.box = "ubuntu/jammy64"
    db1.vm.hostname = "db1.local"
    db1.vm.network "private_network", ip: "192.168.56.20"
  end
end
```

**Команды для конкретной машины:**
```bash
vagrant up web1           # Запустить только web1
vagrant ssh web1          # SSH к web1
vagrant halt db1          # Остановить db1
vagrant status            # Статус всех машин
```

---

## Provisioning (Автоматическая Подготовка)

**Shell скрипт:**
```ruby
config.vm.provision "shell", inline: <<-SHELL
  apt-get update
  apt-get install -y curl wget git
  echo "Docker installed"
SHELL
```

**Из файла:**
```ruby
config.vm.provision "shell", path: "setup.sh"
```

**Ansible:**
```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
  ansible.inventory_path = "hosts.ini"
end
```

**Выполнить provisioning:**
```bash
# При vagrant up (автоматически)
vagrant up

# Повторить provisioning
vagrant provision

# Только для конкретной машины
vagrant provision web1
```

---

## Жизненный Цикл Диаграмма

```
              vagrant init
                   ↓
        +─────────────────────+
        │  Vagrantfile Ready  │
        +─────────────────────+
                   ↓
              vagrant up
                   ↓
        +─────────────────────+
        │ VM Created & Running│
        +─────────────────────+
          ↙          ↓          ↘
    vagrant ssh  vagrant halt  vagrant destroy
       (use)       (pause)       (delete)
        ↙          ↓          ↘
    Working    Running    Deleted
              (paused)

vagrant provision  → переподготовка
vagrant reload     → перезагрузка
```

---

## SSH Подробнее

**Автоматический SSH:**
```bash
# Vagrant управляет SSH подключением
vagrant ssh

# Внутри машины, стандартная оболочка
bash-5.1$
```

**Ручной SSH (если нужно):**
```bash
# Получить SSH конфигурацию
vagrant ssh-config

# Использовать стандартный SSH клиент
ssh -i .vagrant/machines/default/virtualbox/private_key \
    -u vagrant 192.168.56.10 \
    -p 22
```

**Копирование файлов:**
```bash
# На хосте
scp -i .vagrant/machines/default/virtualbox/private_key \
    /local/file vagrant@192.168.56.10:/tmp/

# Или через rsync (если установлен)
vagrant rsync
```

---

## Практический Пример

```bash
# 1. Создать проект
mkdir web-server
cd web-server

# 2. Инициализировать
vagrant init ubuntu/jammy64

# 3. Отредактировать Vagrantfile
vim Vagrantfile
# Добавить:
# - memory 2048, cpus 2
# - port forwarding 8080:8080
# - provision с nginx

# 4. Запустить
vagrant up

# 5. Подключиться и проверить
vagrant ssh
$ curl http://localhost:8080

# 6. Вернуться на хост
$ exit

# 7. Остановить
vagrant halt

# 8. Удалить
vagrant destroy
```

---

**Следующее:** [[04-cluster-setup|Создание Кластера Серверов]]
