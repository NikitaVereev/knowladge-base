---
title: "Ansible CLI Cheatsheet"
type: reference
tags: [ansible, cli, cheatsheet, ansible-playbook, ansible-vault, ansible-galaxy, ansible-inventory]
sources:
  docs: "https://docs.ansible.com/ansible/latest/command_guide/index.html"
related:
  - "[[ansible/reference/modules]]"
  - "[[ansible/reference/jinja2-filters]]"
  - "[[ansible/how-to/write-playbooks]]"
---

# Шпаргалка по Ansible CLI

> **Справочник:** Все команды Ansible на одной странице. Ctrl+F для поиска.

## ansible-playbook

```bash
# Базовый запуск
ansible-playbook site.yml
ansible-playbook -i inventory/production site.yml

# Dry run
ansible-playbook site.yml --check
ansible-playbook site.yml --check --diff

# Ограничение хостов
ansible-playbook site.yml --limit web-01
ansible-playbook site.yml --limit "webservers:!web-03"
ansible-playbook site.yml --limit "webservers:&production"

# Теги
ansible-playbook site.yml --tags "deploy,config"
ansible-playbook site.yml --skip-tags "debug"
ansible-playbook site.yml --list-tags

# Extra vars (наивысший приоритет)
ansible-playbook site.yml -e "version=2.1.0"
ansible-playbook site.yml -e "@vars.json"          # из файла

# Verbosity
ansible-playbook site.yml -v                        # результаты
ansible-playbook site.yml -vv                       # параметры модулей
ansible-playbook site.yml -vvv                      # SSH-соединения

# Vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass

# Sudo
ansible-playbook site.yml -K                        # запросить sudo-пароль
ansible-playbook site.yml -b                        # become (sudo)
ansible-playbook site.yml -u deploy                 # SSH-пользователь

# Инспекция (без выполнения)
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --syntax-check

# Управление выполнением
ansible-playbook site.yml --step                    # подтверждение каждой задачи
ansible-playbook site.yml --start-at-task "Deploy"  # начать с задачи
ansible-playbook site.yml --forks 20                # параллельность
```

## ansible (ad-hoc)

```bash
# Ping (SSH + Python проверка)
ansible all -m ping
ansible webservers -m ping

# Shell-команды
ansible all -m shell -a "uptime"
ansible all -m shell -a "df -h"
ansible all -m command -a "python3 --version"

# Сбор фактов
ansible web-01 -m setup
ansible web-01 -m setup -a "filter=ansible_distribution*"

# Управление файлами
ansible all -m copy -a "src=./file dest=/tmp/"
ansible all -m file -a "path=/tmp/test state=directory"

# Управление пакетами
ansible all -m apt -a "name=htop state=present" -b
ansible all -m apt -a "upgrade=dist update_cache=yes" -b

# Управление сервисами
ansible all -m service -a "name=nginx state=restarted" -b

# С конкретным inventory
ansible -i inventory/staging all -m ping
```

## ansible-vault

```bash
# Создать зашифрованный файл
ansible-vault create secrets.yml

# Редактировать
ansible-vault edit secrets.yml

# Зашифровать существующий файл
ansible-vault encrypt vars.yml

# Расшифровать
ansible-vault decrypt vars.yml

# Просмотреть (без изменения)
ansible-vault view secrets.yml

# Сменить пароль
ansible-vault rekey secrets.yml

# Зашифровать строку (для вставки в YAML)
ansible-vault encrypt_string 'SuperSecret' --name 'db_password'

# Несколько vault ID
ansible-vault encrypt --vault-id prod@prompt secrets.yml
ansible-playbook site.yml --vault-id prod@.vault_pass
```

## ansible-galaxy

```bash
# Роли
ansible-galaxy init roles/myrole                # создать scaffold
ansible-galaxy install geerlingguy.nginx        # установить роль
ansible-galaxy install -r requirements.yml      # из файла зависимостей
ansible-galaxy list                             # установленные роли
ansible-galaxy remove geerlingguy.nginx         # удалить

# Коллекции
ansible-galaxy collection install community.docker
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list

# Поиск
ansible-galaxy search nginx
ansible-galaxy info geerlingguy.nginx
```

## ansible-inventory

```bash
# Граф групп и хостов
ansible-inventory --graph
ansible-inventory -i inventory/production --graph

# Переменные конкретного хоста
ansible-inventory --host web-01

# Полный вывод в JSON
ansible-inventory --list

# YAML
ansible-inventory --list --yaml
```

## ansible-config

```bash
# Показать текущую конфигурацию
ansible-config dump
ansible-config dump --only-changed      # только изменённые

# Список всех параметров
ansible-config list

# Какой конфиг используется
ansible-config view
```

## ansible-doc

```bash
# Документация модуля
ansible-doc ansible.builtin.apt
ansible-doc ansible.builtin.template

# Список модулей
ansible-doc -l                          # все
ansible-doc -l | grep docker            # поиск

# Примеры использования
ansible-doc -s ansible.builtin.apt      # только EXAMPLES

# Плагины
ansible-doc -t lookup file              # lookup-плагин
ansible-doc -t filter to_yaml           # filter-плагин
```

## ansible-lint

```bash
# Проверка
ansible-lint site.yml
ansible-lint roles/
ansible-lint playbooks/ roles/

# С игнорированием правил
ansible-lint -x command-instead-of-shell site.yml

# Показать все правила
ansible-lint -L
```

## Переменные окружения

```bash
# Альтернатива ansible.cfg
export ANSIBLE_CONFIG=/path/to/ansible.cfg
export ANSIBLE_INVENTORY=inventory/production
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
export ANSIBLE_FORKS=20
export ANSIBLE_STDOUT_CALLBACK=yaml
export ANSIBLE_HOST_KEY_CHECKING=False

# Для отладки
export ANSIBLE_DEBUG=1
export ANSIBLE_LOG_PATH=/tmp/ansible.log
```

## Однострочники

```bash
# Ping всех хостов с таймаутом
ansible all -m ping -T 5

# Uptime всех серверов
ansible all -m shell -a "uptime" -o      # -o = одна строка на хост

# Перезагрузить один сервер
ansible web-01 -m reboot -b

# Собрать конкретный факт со всех хостов
ansible all -m setup -a "filter=ansible_memtotal_mb" -o

# Скопировать SSH-ключ
ansible all -m authorized_key -a "user=deploy key='$(cat ~/.ssh/id_ed25519.pub)'" -b

# Обновить все пакеты
ansible all -m apt -a "upgrade=dist update_cache=yes" -b --forks 5

# Выполнить playbook только на хостах, которые изменились
ansible-playbook site.yml --limit @site.retry
```
