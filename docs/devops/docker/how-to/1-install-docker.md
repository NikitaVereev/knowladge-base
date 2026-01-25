---
title: 1 Установка Docker (Linux)
description: Актуальный гайд (2025/26) по установке Docker Engine на Ubuntu 24.04 и Arch Linux. Настройка прав (non-root) и автозапуск.
---

## 1. Ubuntu 22.04 / 24.04 (Debian-based)

Официальная документация: [Install on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Мы используем официальный репозиторий Docker и современный формат `deb822` (`.sources`), который является стандартом для новых версий Ubuntu.

### Шаг 1. Удаление конфликтных пакетов
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

### Шаг 2. Настройка репозитория
Устанавливаем необходимые утилиты и GPG-ключ:

```bash
# 1. Обновляем индекс пакетов
sudo apt-get update
sudo apt-get install ca-certificates curl

# 2. Скачиваем официальный GPG ключ Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Добавляем репозиторий (формат deb822)
echo "Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc" | \
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null

# 4. Обновляем кэш apt
sudo apt-get update
```

### Шаг 3. Установка пакетов
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Шаг 4. Проверка
```bash
sudo docker run hello-world
```

---

## 2. Arch Linux / Manjaro

Wiki: [Docker - ArchWiki](https://wiki.archlinux.org/title/Docker)

В Arch Linux пакеты Docker находятся в официальном репозитории `extra`.

### Шаг 1. Установка
```bash
sudo pacman -Syu
sudo pacman -S docker docker-compose
```

### Шаг 2. Запуск демона
В Arch сервисы не стартуют автоматически после установки.
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 3. Post-installation (Настройка прав)

Документация: [Linux Post-install steps](https://docs.docker.com/engine/install/linux-postinstall/)

По умолчанию демон Docker работает от `root`, поэтому все команды требуют `sudo`. Чтобы работать от своего пользователя:

1.  **Создайте группу** (обычно создается при установке):
    ```bash
    sudo groupadd docker
    ```

2.  **Добавьте себя в группу**:
    ```bash
    sudo usermod -aG docker $USER
    ```

3.  **Примените изменения**:
    Самый надежный способ — выйти из системы (Log Out) и зайти снова.
    Быстрый способ (в текущей сессии):
    ```bash
    newgrp docker
    ```

4.  **Проверка (без sudo)**:
    ```bash
    docker ps
    # Должен вывести пустой список, а не "permission denied".
    ```

## 4. Настройка ротации логов

По умолчанию Docker хранит логи вечно. Это может забить диск за пару недель активной работы.

1.  Откройте (или создайте) файл конфига:
    ```bash
    sudo nano /etc/docker/daemon.json
    ```

2.  Вставьте настройки:
    ```json
    {
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

3.  Перезапустите Docker:
    ```bash
    sudo systemctl restart docker
    ```
