---
title: "1 Установка и настройка Ansible"
description: "Установка Ansible на Ubuntu, настройка инвентаря, конфигурационного файла и проверка соединения с хостами."
---


Ansible работает по модели "Push": вы устанавливаете его только на свой компьютер (Control Node), а на управляемые серверы (Managed Nodes) ничего ставить не нужно, кроме Python и SSH-сервера.

## Установка

Рекомендуется использовать PPA для получения свежей версии, так как в стандартных репозиториях версия может быть устаревшей.

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

Проверка версии:
```bash
ansible --version
```

## Структура проекта

Создайте базовую директорию для конфигурации:
```bash
mkdir -p ~/ansible/{inventory,playbooks,roles}
cd ~/ansible
```

## Настройка инвентаря (Inventory)

Файл инвентаря описывает, какими серверами мы управляем. Создайте файл `inventory/hosts`:

```ini
[local]
localhost ansible_connection=local

[webservers]
# Пример подключения по SSH с указанием пользователя
server1 ansible_host=192.168.1.100 ansible_user=ubuntu
server2 ansible_host=192.168.1.101 ansible_user=ubuntu

[docker_nodes]
server1
server2

# Переменные для группы docker_nodes
[docker_nodes:vars]
ansible_python_interpreter=/usr/bin/python3
```

## Конфигурация (ansible.cfg)

Чтобы не указывать ключи каждый раз в командной строке, создайте файл `ansible.cfg` в корне проекта:

```ini
[defaults]
inventory = inventory/hosts
# Отключаем проверку ключей хоста (удобно для dev-сред, но небезопасно для prod)
host_key_checking = False
# Пользователь по умолчанию на удаленных серверах
remote_user = ubuntu
# Путь к вашему приватному SSH ключу
private_key_file = ~/.ssh/id_ed25519
```

## Проверка соединения

Команда `ping` в Ansible — это не ICMP пинг, а проверка возможности зайти по SSH и выполнить Python-код.

```bash
ansible all -m ping
```

**Успешный результат:**
```json
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Связанные материалы

- [[tools/ssh/how-to/generate-keys|Настройка SSH ключей]]
- [[devops/ansible/tutorials/first-playbook|Написание первого Playbook]]
