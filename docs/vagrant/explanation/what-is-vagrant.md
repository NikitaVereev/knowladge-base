---
title: "Что такое Vagrant"
type: explanation
tags: [vagrant, virtualbox, iac, vm, provider, box, provisioner]
sources:
  original: "devops/vagrant/explanation/1-what-is-vagrant.md + 2-vagrant-vs-others.md"
related:
  - "[[vagrant/explanation/networking]]"
  - "[[vagrant/tutorials/01-first-vm]]"
  - "[[docker/index]]"
---

# Что такое Vagrant

> **TL;DR:** Vagrant = IaC для виртуальных машин. Описываете VM в `Vagrantfile` → `vagrant up` → готовый сервер.
> Не гипервизор, а оркестратор (управляет VirtualBox/VMware). Для инфраструктуры, не для приложений (= Docker).

## Концепция

Vagrant решает проблему «на моём компьютере работает», заменяя ручную настройку VM на конфигурационный файл.

- **Декларативность:** описываете *что* хотите (ОС, RAM, IP), Vagrant делает *как*
- **Воспроизводимость:** один Vagrantfile → идентичные среды на Windows, macOS, Linux
- **Одноразовость:** `vagrant destroy` + `vagrant up` = чистая среда за минуты

## Архитектура

```
User → CLI (vagrant up) → Vagrant Core → Provider (VirtualBox) → VM
                                ↑
                          Vagrantfile
```

| Компонент | Описание | Примеры |
|-----------|---------|---------|
| **Provider** | Гипервизор, запускающий VM | VirtualBox, VMware, Hyper-V, Libvirt |
| **Box** | Предсобранный образ ОС (аналог Docker image) | `ubuntu/noble64`, `generic/arch` |
| **Provisioner** | Настройка VM после запуска | Shell, Ansible, Puppet |

## Vagrant vs Docker

| | Vagrant | Docker |
|---|---------|--------|
| Виртуализирует | Аппаратное обеспечение (полная VM) | Ядро ОС (контейнер) |
| Изоляция | Полная (своё ядро) | Частичная (общее ядро с хостом) |
| Запуск | Минуты | Секунды |
| Ресурсы | Тяжёлый (выделенная RAM) | Лёгкий |
| Цель | Эмуляция **серверов** | Упаковка **приложений** |

**Vagrant** — когда нужна полноценная ОС: тестирование Ansible ролей, сетевых конфигураций, ядра. **Docker** — когда нужен сервис: веб-приложение, БД, очередь.

## Vagrant vs Terraform

Оба от HashiCorp, оба IaC, но разные среды:

- **Vagrant** → локальные среды разработки (VirtualBox на ноутбуке)
- **Terraform** → облачная инфраструктура (AWS, GCP, Azure)

## Зачем использовать

- Тестирование Ansible-плейбуков на «живых» VM
- Моделирование кластера (web + db + cache)
- Разработка с доступом к ядру (недоступно в контейнерах)
- Изоляция legacy-проектов (старые стеки)
- Обучение системному администрированию