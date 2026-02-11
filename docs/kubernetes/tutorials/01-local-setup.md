---
title: "01 — Настройка локального окружения"
type: tutorial
tags: [kubernetes, tutorial, minikube, kubectl, setup, docker-driver, k9s]
sources:
  docs: "https://minikube.sigs.k8s.io/docs/start/"
related:
  - "[[kubernetes/explanation/architecture]]"
  - "[[kubernetes/tutorials/02-first-deployment]]"
---

# Tutorial 01 — Настройка локального окружения

> **Цель:** Установить kubectl + Minikube, запустить одноузловой кластер, проверить работу.
> Добавить полезные утилиты (k9s, autocompletion).

**Время:** ~20 минут
**Требования:** Docker Engine установлен, пользователь в группе `docker`.

## Шаг 1. Установка kubectl

```bash
# Arch Linux
sudo pacman -S kubectl

# Ubuntu / Debian
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# macOS
brew install kubectl

# Проверка
kubectl version --client --output=yaml
```

## Шаг 2. Установка Minikube

```bash
# Arch Linux
sudo pacman -S minikube

# Ubuntu / Debian
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb

# macOS
brew install minikube
```

## Шаг 3. Запуск кластера

```bash
# Запуск с Docker-драйвером (рекомендуется)
minikube start --driver=docker

# С дополнительными ресурсами (для production-like нагрузок)
minikube start --driver=docker --cpus=4 --memory=4096 --disk-size=20g
```

**Что происходит:**
1. Скачивается образ `kicbase` (Kubernetes in Container)
2. Создаётся контейнер, эмулирующий ноду
3. Генерируется `~/.kube/config` с контекстом `minikube`

## Шаг 4. Проверка

```bash
# Список нод
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.31.0

# Системные поды (Control Plane)
kubectl get pods -A
# kube-system   coredns-...          Running
# kube-system   etcd-minikube        Running
# kube-system   kube-apiserver-...   Running

# Информация о кластере
kubectl cluster-info

# Контекст (должен быть minikube)
kubectl config current-context
```

## Шаг 5. Полезные addons

```bash
# Включить Ingress-контроллер (Nginx)
minikube addons enable ingress

# Включить metrics-server (для kubectl top)
minikube addons enable metrics-server

# Dashboard
minikube addons enable dashboard
minikube dashboard    # откроет в браузере

# Список всех addons
minikube addons list
```

## Шаг 6. Дополнительные утилиты

### Autocompletion (kubectl)

```bash
# Bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc

source ~/.bashrc  # или ~/.zshrc
```

### k9s (TUI для Kubernetes)

```bash
# Arch
sudo pacman -S k9s

# Homebrew
brew install derailed/k9s/k9s

# Запуск
k9s
```

k9s — терминальный интерфейс: навигация по ресурсам, логи, exec в поды, удаление — всё без kubectl.

## Управление кластером

```bash
# Остановить (сохранить состояние)
minikube stop

# Запустить снова
minikube start

# Удалить кластер
minikube delete

# Несколько кластеров (profiles)
minikube start -p dev-cluster
minikube start -p test-cluster
kubectl config get-contexts      # увидите оба
```

## Troubleshooting

| Проблема | Решение |
|----------|---------|
| `permission denied ... Docker daemon socket` | `sudo usermod -aG docker $USER && newgrp docker` |
| `GUEST_MISSING_CONNTRACK` (Arch) | `sudo pacman -S conntrack-tools` |
| `driver failed` / зависает | `minikube delete && minikube start --driver=docker` |
| kubectl показывает старый кластер | `kubectl config use-context minikube` |
| Нехватка ресурсов | `minikube start --cpus=4 --memory=4096` |

## Что дальше

→ [[kubernetes/tutorials/02-first-deployment]] — Deployment, Service, доступ к приложению

## Альтернативы Minikube

| Инструмент | Особенности | Когда использовать |
|-----------|-------------|-------------------|
| **Minikube** | Полнофункциональный, addons, multi-profile | Обучение, локальная разработка |
| **kind** (K8s in Docker) | Легковесный, multi-node, CI/CD | Тестирование, CI |
| **k3s** | Минимальный K8s, production-ready | Edge, IoT, малые серверы |
| **Docker Desktop** | Встроенный K8s | macOS/Windows, простой setup |