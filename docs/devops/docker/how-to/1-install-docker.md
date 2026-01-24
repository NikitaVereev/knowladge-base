---
title: "1 Установка Docker на Ubuntu 24.04"
description: "Актуальное руководство (2025/26) по установке Docker Engine, настройке Rootless режима и плагина Compose v2."
---

В Ubuntu 24.04 `docker.io` из стандартных репозиториев может быть не самой свежей версии. Рекомендуется ставить **Docker CE (Community Edition)** из официального репозитория Docker.

## Шаг 1: Подготовка системы

Удалите старые или конфликтующие пакеты (если они были):
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

## Шаг 2: Установка репозитория

1.  Обновите индексы и поставьте зависимости:
    ```bash
    sudo apt-get update
    sudo apt-get install ca-certificates curl
    ```

2.  Добавьте официальный GPG-ключ Docker:
    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

3.  Добавьте репозиторий в источники `apt`:
    ```bash
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt-get update
    ```

## Шаг 3: Установка Docker Engine

Установите сам движок, CLI, containerd и плагины (Buildx и Compose):
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверьте, что демон запущен:
```bash
sudo systemctl status docker
```

## Шаг 4: Настройка прав (Post-install)

По умолчанию Docker работает только через `sudo`. Чтобы запускать его от вашего пользователя:

1.  Создайте группу `docker` (обычно создается при установке) и добавьте себя в нее:
    ```bash
    sudo usermod -aG docker $USER
    ```

2.  **Важно:** Примените изменения групп. Либо перезайдите в систему (Logout/Login), либо выполните:
    ```bash
    newgrp docker
    ```

3.  Проверьте работу без sudo:
    ```bash
    docker run hello-world
    ```

## Полезные настройки (Log Rotation)
По умолчанию Docker хранит логи вечно, что может забить диск. Создайте файл `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
После этого: `sudo systemctl restart docker`.
