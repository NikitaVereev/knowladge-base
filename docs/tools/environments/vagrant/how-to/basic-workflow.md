---
title: "Базовый рабочий процесс Vagrant"
description: "Пошаговый workflow: инициализация, запуск, подключение, остановка и удаление виртуальных машин."
---


Стандартный цикл работы с виртуальной машиной в Vagrant состоит из 7 шагов.

## 1. Инициализация проекта

Создайте директорию и инициализируйте Vagrantfile:

```bash
mkdir my-vagrant-project
cd my-vagrant-project
vagrant init ubuntu/noble64  # Ubuntu 24.04 LTS
```

Это создаст файл `Vagrantfile` с базовой конфигурацией.

## 2. Редактирование конфигурации

Откройте `Vagrantfile` и настройте ресурсы:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/noble64"
  
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end
  
  # Проброс порта (опционально)
  config.vm.network "forwarded_port", guest: 80, host: 8080
end
```

## 3. Запуск ВМ

```bash
vagrant up
```

При первом запуске:
- Скачивает box (~600 MB для Ubuntu)
- Создаёт ВМ в VirtualBox
- Запускает ОС
- Настраивает SSH-доступ

Время: 5-10 минут при первом запуске, 1-2 минуты при последующих.

## 4. Подключение по SSH

```bash
vagrant ssh
```

Вы попадёте внутрь ВМ под пользователем `vagrant` (sudo без пароля).

```bash
vagrant@ubuntu:~$ uname -a
Linux ubuntu 6.8.0-51-generic #52-Ubuntu SMP x86_64 GNU/Linux

vagrant@ubuntu:~$ exit  # Выход из ВМ
```

## 5. Тестирование сервисов

Запустите веб-сервер внутри ВМ:

```bash
vagrant ssh
python3 -m http.server 8080
```

На хосте (другой терминал):
```bash
curl http://localhost:8080
# Должен вернуть HTML страницу
```

## 6. Остановка ВМ

```bash
vagrant halt
```

Корректно выключает ВМ (как `shutdown -h now`). Данные на диске сохраняются.

## 7. Удаление ВМ

```bash
vagrant destroy -f
```

Удаляет ВМ и освобождает диск. Box остаётся в кэше (можно пересоздать ВМ без повторной загрузки).

## Дополнительные команды

**Перезагрузка с применением нового Vagrantfile:**
```bash
vagrant reload
```

**Проверка статуса:**
```bash
vagrant status          # Текущий проект
vagrant global-status   # Все ВМ на хосте
```

**Выполнение provisioners без перезагрузки:**
```bash
vagrant provision
```

**Упаковка ВМ в box:**
```bash
vagrant package --output my-custom-box.box
```

## Типичные проблемы

**ВМ не запускается (VT-x ошибка)**
- Включите виртуализацию в BIOS (Intel VT-x / AMD-V)
- На Windows отключите Hyper-V: `bcdedit /set hypervisorlaunchtype off`

**SSH таймаут при `vagrant up`**
- Проверьте, что VirtualBox установлен корректно
- Переустановите VirtualBox Guest Additions: `vagrant plugin install vagrant-vbguest`

**Медленная работа ВМ**
- Выделите больше RAM в Vagrantfile (`vb.memory = 4096`)
- Используйте SSD для хранения ВМ

## Связанные материалы
- [[tools/environments/vagrant/explanation/what-is-vagrant|Что такое Vagrant]]
- [[tools/environments/vagrant/how-to/vagrantfile-configuration|Продвинутая настройка Vagrantfile]]
- [[tools/environments/vagrant/how-to/multi-machine-setup|Создание кластера из нескольких ВМ]]
