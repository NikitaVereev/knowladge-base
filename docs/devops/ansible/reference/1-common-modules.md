---
title: "1 Популярные модули Ansible"
description: "Шпаргалка по модулям, которые используются в 90% задач."
---

## Управление пакетами
*   **apt:** `name=nginx state=present update_cache=yes` (Debian/Ubuntu)
*   **dnf/yum:** `name=git state=latest` (RHEL/CentOS)
*   **pip:** `name=docker state=present` (Python)
*   **npm:** `name=pm2 global=yes` (Node.js)

## Файлы и Директории
*   **file:** `path=/opt/app state=directory mode='0755'` (создать папку, изменить права)
*   **copy:** `src=files/conf dest=/etc/conf` (статический файл)
*   **template:** `src=templates/conf.j2 dest=/etc/conf` (Jinja2 шаблон)
*   **lineinfile:** `path=/etc/hosts line='1.2.3.4 myserver'` (добавить строку)
*   **unarchive:** `src=app.tar.gz dest=/opt/ remote_src=yes` (распаковка)

## Сервисы и Команды
*   **service / systemd:** `name=nginx state=restarted enabled=yes`
*   **command:** `cmd=/bin/myscript` (безопасный запуск)
*   **shell:** `cmd="ls | grep log"` (поддержка пайпов и редиректов)

## Управление
*   **user:** `name=deploy groups=docker shell=/bin/bash`
*   **git:** `repo=https://github.com/a/b.git dest=/opt/app version=main`
*   **get_url:** `url=https://example.com/file.tar.gz dest=/tmp/` (скачать файл)
