---
title: "Ресурсы и лимиты"
type: how-to
tags: [kubernetes, resources, limits, requests, qos, limitrange, resourcequota, oom]
sources:
  docs: "https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/"
related:
  - "[[kubernetes/how-to/manage-workloads]]"
  - "[[kubernetes/how-to/troubleshoot]]"
  - "[[kubernetes/reference/yaml-templates]]"
---

# Ресурсы и лимиты

> **TL;DR:** `requests` — гарантированный минимум (влияет на scheduling).
> `limits` — жёсткий потолок (превышение CPU = throttling, RAM = OOMKill).
> Всегда задавайте оба. LimitRange/ResourceQuota — защита на уровне namespace.

## Requests и Limits

```yaml
spec:
  containers:
    - name: app
      image: myapp:v1
      resources:
        requests:                  # Scheduler гарантирует эти ресурсы
          cpu: "250m"              # 0.25 ядра (1000m = 1 core)
          memory: "128Mi"          # 128 мебибайт
        limits:                    # Жёсткий потолок
          cpu: "500m"
          memory: "256Mi"
```

### Единицы измерения

| Ресурс | Единицы | Примеры |
|--------|---------|---------|
| CPU | millicores | `100m` = 0.1 ядра, `1` = 1 ядро, `1500m` = 1.5 ядра |
| Memory | bytes | `128Mi` = 128 MiB, `1Gi` = 1 GiB, `256M` = 256 MB (десятичный) |

### Что происходит при превышении

| Ресурс | Действие | Результат |
|--------|----------|-----------|
| CPU > limit | **Throttling** | Под замедляется, но не убивается |
| Memory > limit | **OOMKill** | Контейнер убивается и перезапускается |
| Memory > node capacity | **Eviction** | Kubelet вытесняет поды с ноды |

## QoS классы

Kubernetes автоматически назначает QoS-класс на основе requests/limits.

| QoS Class | Условие | Приоритет при eviction |
|-----------|---------|----------------------|
| **Guaranteed** | requests == limits (для всех контейнеров) | Последний (самый защищённый) |
| **Burstable** | requests < limits (хотя бы у одного) | Средний |
| **BestEffort** | Нет requests и limits | Первый (убивается первым) |

```yaml
# Guaranteed (рекомендуется для production)
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"           # == requests
    memory: "256Mi"       # == requests
```

## Рекомендации по значениям

```yaml
# Начальные значения (подстроить после мониторинга)
# Веб-сервер (Node.js/Python)
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }

# Java-приложение
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }

# PostgreSQL
resources:
  requests: { cpu: "250m", memory: "256Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }

# Nginx sidecar
resources:
  requests: { cpu: "50m", memory: "32Mi" }
  limits:   { cpu: "100m", memory: "64Mi" }
```

## Мониторинг реального потребления

```bash
# Требуется metrics-server
kubectl top pods
kubectl top pods --sort-by=memory
kubectl top nodes

# Minikube
minikube addons enable metrics-server
```

## LimitRange (дефолты для namespace)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: default
spec:
  limits:
    - type: Container
      default:                     # limits по умолчанию
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:              # requests по умолчанию
        cpu: "100m"
        memory: "128Mi"
      max:                         # максимум для контейнера
        cpu: "2"
        memory: "2Gi"
      min:                         # минимум
        cpu: "50m"
        memory: "32Mi"
```

## ResourceQuota (лимит на namespace)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"             # суммарно 10 ядер requests
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"                     # максимум 50 подов
    services: "10"
    persistentvolumeclaims: "20"
    configmaps: "50"
    secrets: "50"
```

```bash
kubectl get resourcequota -n team-a
kubectl describe resourcequota team-quota -n team-a
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Нет requests/limits | Pod = BestEffort, убивается первым при нехватке ресурсов | Всегда указывать requests и limits |
| `requests.memory` слишком мал | Pod не влезает ни на одну ноду → `Pending` | `kubectl describe pod` → Events: `Insufficient memory` |
| `limits.memory` слишком мал | OOMKilled → CrashLoopBackOff | `kubectl top pod`, увеличить limits |
| CPU limits для latency-sensitive | Throttling вызывает всплески задержки | Для latency: убрать CPU limits, оставить только requests |
| Забыли LimitRange | Поды без limits потребляют все ресурсы ноды | Создать LimitRange с defaults для каждого namespace |