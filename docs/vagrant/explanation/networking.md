---
title: "Сетевая модель Vagrant"
type: explanation
tags: [vagrant, networking, nat, private-network, public-network, port-forwarding]
sources:
  original: "devops/vagrant/explanation/3-networking-concepts.md"
related:
  - "[[vagrant/explanation/what-is-vagrant]]"
  - "[[vagrant/how-to/configure-vagrantfile]]"
---

# Сетевая модель Vagrant

> **TL;DR:** NAT + port forwarding (по умолчанию, доступ через localhost:PORT).
> Private network — статический IP, только хост ↔ VM. Public network — VM в вашей LAN.

## 1. NAT + Port Forwarding (default)

VM за NAT: имеет доступ в интернет, но извне недоступна. Доступ к сервисам через проброс портов.

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080, auto_correct: true
```

```
localhost:8080 → VM:80
```

**Сценарий:** Веб-разработка. Сайт в VM на порту 80, открываете `localhost:8080` в браузере.

## 2. Private Network (Host-only)

Виртуальный коммутатор: только хост и VM. Статические IP, недоступны из интернета.

```ruby
config.vm.network "private_network", ip: "192.168.56.10"
```

**Сценарий:** Кластер (web + db). Web подключается к БД по постоянному IP `192.168.56.11`. Безопасно — доступ только с вашего компьютера.

## 3. Public Network (Bridged)

VM подключается к физическому роутеру как отдельное устройство. Получает IP от DHCP.

```ruby
config.vm.network "public_network"
```

**Сценарий:** Тестирование с мобильного телефона в той же WiFi-сети.

**⚠️ Безопасность:** VM видна всем в сети. Не использовать в публичных сетях.

## Сравнение

| Тип | IP | Доступ извне | Безопасность | Сценарий |
|-----|-----|-------------|-------------|---------|
| NAT + forwarding | localhost:PORT | Нет | Высокая | Веб-разработка |
| Private network | 192.168.56.x | Нет (только хост) | Высокая | Кластеры, Ansible |
| Public network | от DHCP роутера | Да (вся LAN) | Низкая | Тестирование с устройств |