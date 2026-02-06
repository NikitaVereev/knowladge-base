---
title: 1 Настройка локального окружения
type: tutorial
tags: [kubernetes, k8s, setup, arch, ubuntu]
---

В этом уроке вы настроите локальный кластер Kubernetes на базе **Minikube**. Это позволит эмулировать полноценную инфраструктуру на одной машине для разработки и тестирования.

**Цель:** Установить инструменты CLI и запустить одноузловой кластер (Single Node Cluster).

## Предварительные требования
*   Установленный Docker Engine (см. [[install|1-install-docker]]).
*   Текущий пользователь добавлен в группу `docker`.

---

## Шаг 1: Установка kubectl

`kubectl` — это консольная утилита (клиент) для управления кластером Kubernetes.

### Arch Linux
Пакет доступен в официальном репозитории `extra`.

```bash
sudo pacman -S kubectl
```

### Ubuntu / Debian
Установка бинарного файла напрямую:

```bash
# 1. Скачивание последней стабильной версии
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 2. Установка
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Проверьте установку (клиентская часть):
```bash
kubectl version --client --output=yaml
```

---

## Шаг 2: Установка Minikube

Minikube запускает кластер Kubernetes внутри Docker-контейнера.

### Arch Linux
Пакет также доступен в `extra`.

```bash
sudo pacman -S minikube
```

### Ubuntu / Debian
Используем `.deb` пакет:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

---

## Шаг 3: Запуск кластера

Инициализируйте кластер, принудительно указав драйвер Docker (наиболее стабильный вариант для локальной разработки).

```bash
minikube start --driver=docker
```

**Что происходит:**
1.  Скачивается образ `kicbase` (Kubernetes in Container).
2.  Создается контейнер, эмулирующий ноду.
3.  Генерируются сертификаты и конфиг `~/.kube/config`.

---

## Шаг 4: Проверка связи

Убедитесь, что `kubectl` автоматически переключился на контекст Minikube.

1.  **Проверка узлов:**
    ```bash
    kubectl get nodes
    ```
    *Ожидаемый вывод:*
    ```text
    NAME       STATUS   ROLES           AGE   VERSION
    minikube   Ready    control-plane   1m    v1.34.0
    ```

2.  **Проверка системных подов:**
    ```bash
    kubectl get pods -A
    ```
    Вы увидите запущенные компоненты Control Plane (etcd, coredns, apiserver).

---

## Troubleshooting (Возможные проблемы)

### 1. Ошибка "permission denied while trying to connect to the Docker daemon socket"
Minikube не может достучаться до Docker.
**Решение:** Убедитесь, что ваш пользователь в группе docker и **перезайдите в сессию** (logout/login).
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Ошибка "Exiting due to GUEST_MISSING_CONNTRACK" (Arch Linux)
На Arch Linux часто отсутствует утилита `conntrack`, необходимая для сетевого стека Kubernetes.
**Решение:**
```bash
sudo pacman -S conntrack-tools
```

### 3. Ошибка "driver failed" или зависание на "Pulling base image"
Если Docker работает нестабильно, попробуйте сбросить кластер:
```bash
minikube delete
minikube start --driver=docker --alsologtostderr
```

---

## Следующий шаг
Кластер готов. Теперь мы развернем в нем ваше первое приложение.

Перейти к уроку: [[2-first-pod|2. Первый Pod (Manifests, Kubectl run)]]
