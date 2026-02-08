---
title: "Установка Docker"
type: how-to
tags: [docker, install, rootless, hardening, ubuntu, arch-linux]
sources:
  docs: "https://docs.docker.com/engine/install/"
related:
  - "[[docker/explanation/architecture]]"
  - "[[docker/tutorials/01-first-container]]"
  - "[[docker/reference/cheatsheet]]"
---

# Установка Docker

> **TL;DR:** Два варианта: Rootless (безопаснее, для рабочих станций) и Rootful + Hardening
> (для серверов). На Ubuntu — официальный репозиторий Docker, не snap.

В этом руководстве рассматриваются два сценария установки:
1.  **Rootless Mode:** Демон и контейнеры работают от обычного пользователя. Идеально для рабочих станций и безопасных CI-агентов.
2.  **Rootful + Hardening:** Классический системный демон с усиленной безопасностью. Для production-серверов, где требуются привилегированные порты (<1024) или специфические драйверы.

---

## Вариант 1: Rootless Mode (Без прав root)

В этом режиме `dockerd` работает внутри User Namespace. Злоумышленник, сбежавший из контейнера, не получит прав root на хосте.

### 1. Подготовка (Prerequisites)

Вам потребуется пакет `uidmap` и наличие диапазонов subuid/subgid.

**Ubuntu 24.04+:**
```bash
sudo apt-get update && sudo apt-get install -y uidmap
```

**Arch Linux:**
```bash
sudo pacman -Syu docker docker-compose
# В Arch сам пакет docker не запускает демон, он нужен только для CLI и скриптов.
```

**Проверка subuid (Важно!):**
Убедитесь, что для вашего пользователя есть записи (обычно они создаются автоматически при создании пользователя):
```bash
grep "^$(whoami):" /etc/subuid
grep "^$(whoami):" /etc/subgid
# Вывод должен быть вида: user:100000:65536
```
*Если вывод пуст, добавьте диапазоны вручную:*
```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)
```

### 2. Установка (Скрипт)

Не используйте `sudo`!

```bash
# Остановите системный докер, если он был запущен
sudo systemctl disable --now docker.service docker.socket

# Запуск скрипта установки
curl -fsSL https://get.docker.com/rootless | sh
```

### 3. Настройка окружения

Скрипт подскажет переменные, которые нужно добавить в `~/.bashrc` или `~/.zshrc`. Обычно это:

```bash
export PATH=/home/$(whoami)/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

Примените изменения: `source ~/.bashrc`

### 4. Запуск и Автозагрузка

```bash
systemctl --user start docker
systemctl --user enable docker

# Чтобы демон запускался при рестарте сервера (даже без логина пользователя)
sudo loginctl enable-linger $(whoami)
```

---

## Вариант 2: Rootful Mode (Системный демон)

Используйте этот вариант, если вам нужен доступ к системным ресурсам или вы управляете кластером через Ansible/Chef.

### 1. Установка

#### Ubuntu 24.04 / Debian
Официальная документация: [Install on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Удаляем конфликтующие пакеты и ставим официальную версию.

```bash
# Удаление старых версий
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

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

# Установка
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Arch Linux
Wiki: [Docker - ArchWiki](https://wiki.archlinux.org/title/Docker)

В Arch Linux пакеты Docker находятся в официальном репозитории `extra`.

```bash
sudo pacman -Syu
sudo pacman -S docker docker-compose
```

### 2. Hardening

Не добавляйте пользователя в группу `docker` бездумно! Вместо этого настройте демона.

#### Конфигурация `/etc/docker/daemon.json`
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "icc": false
}
```
* `log-opts`: Ограничивает размер лог-файл 10Мб и хранит 3 последние копии.
* `"no-new-privileges": true`: Блокирует setuid-бинарники внутри контейнеров.
* `live-restore`: Позволяет контейнерам работать, даже если демон Docker перезагружается (обновляется)
* `userland-proxy`: Отключает прокси-процесс для проброса портов (использует только iptables), экономя память.
* `"icc": false`: Запрещает контейнерам в дефолтной сети `bridge` общаться друг с другом (изоляция).

Примените: `sudo systemctl restart docker`

---

## Проверка установки

Независимо от метода, проверьте версию и контекст безопасности.

```bash
docker info
# Ищите строку "Security Options".
# В Rootless там будет "rootless".
# В Rootful ищите "seccomp", "apparmor".

docker run --rm hello-world
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Установка Docker из snap | Старая версия, проблемы с путями | Удалить snap, поставить из apt-репозитория Docker |
| Добавили пользователя в группу docker | Работает, но это эквивалент root-доступа | Использовать Rootless mode для рабочих станций |
| Не включили lingering для rootless | Docker rootless останавливается при logout | `loginctl enable-linger $USER` |
| Docker не стартует после reboot | Сервис не включён | `systemctl enable docker` (rootful) или `systemctl --user enable docker` (rootless) |