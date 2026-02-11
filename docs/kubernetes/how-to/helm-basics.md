---
title: "Helm (Package Manager)"
type: how-to
tags: [kubernetes, helm, chart, release, values, repository, upgrade, rollback]
sources:
  docs: "https://helm.sh/docs/"
related:
  - "[[kubernetes/how-to/manage-workloads]]"
  - "[[kubernetes/how-to/recipes/monitoring]]"
  - "[[kubernetes/reference/yaml-templates]]"
---

# Helm — Package Manager для Kubernetes

> **TL;DR:** Helm = `apt` для Kubernetes. Chart = пакет с шаблонами YAML.
> `helm install` — установить, `helm upgrade` — обновить, `helm rollback` — откатить.
> `values.yaml` — настройка чарта без изменения шаблонов.

## Установка

```bash
# Homebrew (macOS/Linux)
brew install helm

# Arch Linux
sudo pacman -S helm

# Скрипт
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Проверка
helm version
```

## Основные команды

### Работа с репозиториями

```bash
# Добавить популярные репозитории
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Обновить индекс
helm repo update

# Поиск чартов
helm search repo nginx
helm search repo postgres
helm search hub grafana           # поиск в Artifact Hub
```

### Установка и управление

```bash
# Установить (release name = my-nginx)
helm install my-nginx bitnami/nginx

# Установить с кастомными values
helm install my-nginx bitnami/nginx -f values.yaml
helm install my-nginx bitnami/nginx --set replicaCount=3

# В конкретный namespace
helm install my-nginx bitnami/nginx -n web --create-namespace

# Dry-run (посмотреть что будет создано)
helm install my-nginx bitnami/nginx --dry-run --debug

# Показать сгенерированные манифесты
helm template my-nginx bitnami/nginx -f values.yaml
```

### Обновление и откат

```bash
# Обновить release
helm upgrade my-nginx bitnami/nginx -f values.yaml
helm upgrade my-nginx bitnami/nginx --set image.tag=1.25

# Upgrade или install (идемпотентно)
helm upgrade --install my-nginx bitnami/nginx -f values.yaml

# История версий
helm history my-nginx

# Откат к предыдущей версии
helm rollback my-nginx
helm rollback my-nginx 2          # к конкретной ревизии
```

### Управление releases

```bash
# Список установленных
helm list
helm list -A                      # все namespaces

# Информация о release
helm status my-nginx
helm get values my-nginx          # текущие values
helm get manifest my-nginx        # сгенерированные YAML

# Удалить
helm uninstall my-nginx
```

## values.yaml

Параметры для настройки чарта. Переопределяют `defaults`.

```bash
# Посмотреть доступные параметры
helm show values bitnami/nginx > values.yaml
```

Пример `values.yaml`:

```yaml
# values.yaml для bitnami/nginx
replicaCount: 3

image:
  tag: "1.25"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  hostname: app.example.com
  tls: true

persistence:
  enabled: false
```

### Приоритет values

```bash
# 1. Значения из чарта (defaults)
# 2. -f values.yaml (файл)
# 3. --set key=value (командная строка, наивысший приоритет)

helm install my-app chart/ \
  -f values-base.yaml \
  -f values-production.yaml \
  --set image.tag=v1.5.0
```

## Популярные чарты

```bash
# Ingress-контроллер (Nginx)
helm install ingress ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace

# Cert-manager (Let's Encrypt)
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace \
  --set installCRDs=true

# Prometheus + Grafana
helm install monitoring prometheus/kube-prometheus-stack -n monitoring --create-namespace

# PostgreSQL
helm install db bitnami/postgresql --set auth.postgresPassword=secret

# Redis
helm install cache bitnami/redis --set auth.enabled=false
```

## Создание своего чарта

```bash
# Scaffold
helm create my-app

# Структура
my-app/
├── Chart.yaml                # metadata (name, version, description)
├── values.yaml               # default values
├── charts/                   # зависимости (subchart)
├── templates/                # шаблоны K8s-манифестов
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl          # вспомогательные шаблоны
│   └── NOTES.txt             # сообщение после установки
└── .helmignore

# Проверить шаблоны
helm template my-app ./my-app -f values.yaml

# Lint
helm lint ./my-app

# Упаковать
helm package ./my-app         # → my-app-0.1.0.tgz

# Установить из локальной директории
helm install my-release ./my-app -f values.yaml
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `helm repo update` не выполнен | Старая версия чарта | Всегда `helm repo update` перед install |
| `--set` для сложных значений | Синтаксис ломается | Использовать `-f values.yaml` для сложных конфигов |
| Забыли `--create-namespace` | `namespace not found` | `helm install ... -n ns --create-namespace` |
| Values не переопределяют defaults | Опечатка в ключе | `helm show values chart/` для проверки структуры |
| `helm uninstall` не удалил PVC | Данные остались | PVC не удаляются при uninstall. `kubectl delete pvc ...` вручную |
