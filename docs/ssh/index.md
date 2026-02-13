---
title: "SSH"
type: index
tags: [ssh, security, remote, keys, tunnels]
---

# SSH

Безопасный протокол удалённого доступа. Шифрованные подключения, аутентификация по ключам, туннели, передача файлов.

## Explanation

| Документ | Описание |
|----------|----------|
| [[ssh/explanation/how-ssh-works]] | Протокол, шифрование, аутентификация по ключам, алгоритмы |

## Tutorials

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[ssh/tutorials/01-getting-started]] | Генерация Ed25519, ssh-copy-id, подключение, ssh-agent |

## How-to

| Документ | Описание |
|----------|----------|
| [[ssh/how-to/configure-client]] | `~/.ssh/config`: алиасы, jump hosts, multiplexing |
| [[ssh/how-to/harden-server]] | sshd_config: отключить пароли, fail2ban, ограничить доступ |
| [[ssh/how-to/tunnels]] | Local/Remote/Dynamic forwarding, SOCKS proxy |

## Recipes

| Рецепт | Описание |
|--------|----------|
| [[ssh/how-to/recipes/ssh-config-examples]] | Готовые блоки config: bastion, GitHub, Vagrant, tunnels |

## Reference

| Документ | Описание |
|----------|----------|
| [[ssh/reference/cheatsheet]] | ssh, scp, rsync, ssh-keygen, agent, права доступа |

## Быстрый старт

```bash
# Генерация ключа
ssh-keygen -t ed25519 -C "email@example.com"

# Копирование на сервер
ssh-copy-id user@server

# Подключение
ssh user@server
```

## Связанные разделы

- [[linux/how-to/recipes/ssh-hardening]] — SSH hardening в контексте Linux-сервера
- [[vagrant/index]] — Vagrant (SSH к виртуальным машинам)
- [[ansible/index]] — Ansible (работает поверх SSH)
