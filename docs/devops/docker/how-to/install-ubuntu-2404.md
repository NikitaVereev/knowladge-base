---
title: "Установка Docker на Ubuntu 24.04"
description: "Пошаговая инструкция по установке Docker Engine и Docker Compose на Ubuntu 24.04 LTS из официального репозитория."
---


Официальный репозиторий Docker обеспечивает установку самой свежей версии Engine, в отличие от стандартных репозиториев Ubuntu.

## Подготовка системы

1. **Удалите старые версии (конфликтующие пакеты):**
   ```bash
   sudo apt remove docker.io docker-compose docker-compose-v2 docker-doc podman-docker
   ```

2. **Установите необходимые утилиты:**
   ```bash
   sudo apt update
   sudo apt install -y ca-certificates curl
   ```

## Настройка репозитория

1. **Добавьте GPG ключ Docker:**
   ```bash
   sudo install -m 0755 -d /etc/apt/keyrings
   sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
   sudo chmod a+r /etc/apt/keyrings/docker.asc
   ```

2. **Добавьте репозиторий в источники:**
   ```bash
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   ```

## Установка Engine

1. **Установите пакеты:**
   ```bash
   sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
   *Примечание:* Команда устанавливает Engine, CLI, Containerd и плагин Compose (теперь это часть пакета `docker-compose-plugin`, команда `docker compose`, а не `docker-compose`).

2. **Проверьте статус службы:**
   ```bash
   sudo systemctl status docker
   ```
   Должен быть статус `active (running)`.

## Настройка прав доступа (rootless mode)

По умолчанию Docker требует `sudo`. Чтобы запускать команды от обычного пользователя:

1. **Добавьте пользователя в группу `docker`:**
   ```bash
   sudo usermod -aG docker $USER
   ```

2. **Примените изменения:**
   Либо перезайдите в систему (logout/login), либо выполните:
   ```bash
   newgrp docker
   ```

3. **Проверьте работу:**
   ```bash
   docker run hello-world
   ```
   Если вы видите сообщение "Hello from Docker!", установка завершена успешно.

## Связанные материалы

- [[devops/docker/explanation/architecture|Архитектура Docker]]
- [[devops/ansible/tutorials/install-docker-playbook|Установка Docker через Ansible]]
