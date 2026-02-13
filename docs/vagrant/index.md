---
title: "Vagrant"
type: index
tags: [vagrant, virtualbox, vm, iac, development-environment]
---

# Vagrant

IaC для виртуальных машин. Описываете VM в `Vagrantfile` → `vagrant up` → готовый сервер. Оркестратор поверх VirtualBox/VMware.

## Explanation

| Документ | Описание |
|----------|----------|
| [[vagrant/explanation/what-is-vagrant]] | Концепция, архитектура, Provider/Box/Provisioner, сравнение с Docker |
| [[vagrant/explanation/networking]] | NAT, Private Network, Public Network, port forwarding |

## Tutorials

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[vagrant/tutorials/01-first-vm]] | Установка VirtualBox + Vagrant, первая VM, жизненный цикл |
| 02 | [[vagrant/tutorials/02-docker-cluster]] | 3 VM с Docker, Private Network, Shell provisioning |

## How-to

| Документ | Описание |
|----------|----------|
| [[vagrant/how-to/configure-vagrantfile]] | CPU/RAM, сеть, synced folders, SSH |
| [[vagrant/how-to/provisioning]] | Shell inline/external, Ansible, порядок запуска |
| [[vagrant/how-to/multi-machine]] | Кластер: циклы, разные роли, управление нодами |
| [[vagrant/how-to/ansible-integration]] | Ansible provisioner, inventory, запуск вручную |

## Reference

| Документ | Описание |
|----------|----------|
| [[vagrant/reference/cli-commands]] | up, halt, ssh, provision, snapshot, box, global-status |

## Быстрый старт

```bash
mkdir project && cd project
vagrant init ubuntu/noble64
vagrant up
vagrant ssh
# Внутри VM...
exit
vagrant destroy -f
```

## Связанные разделы

- [[ssh/index]] — SSH (подключение к VM)
- [[ansible/index]] — Ansible (автоматизация настройки VM)
- [[docker/index]] — Docker (контейнеризация вместо VM)
