---
title: "Декларативный подход и манифесты"
type: explanation
tags: [kubernetes, manifests, yaml, declarative, imperative, apply, gitops, idempotent]
sources:
  docs: "https://kubernetes.io/docs/concepts/overview/working-with-objects/"
related:
  - "[[kubernetes/explanation/architecture]]"
  - "[[kubernetes/explanation/workload-resources]]"
  - "[[kubernetes/reference/yaml-templates]]"
---

# Декларативный подход и манифесты

> **TL;DR:** Императивный (`kubectl create`) — для тестов. Декларативный (`kubectl apply -f`) — для production.
> Манифест = YAML-файл с desired state. Apply идемпотентен: запускай сколько угодно раз.
> Всё в Git → GitOps.

В Kubernetes два подхода к управлению ресурсами. Понимание разницы — ключ к переходу от «попробовать» к production.

## Императивный подход

«Сделай это прямо сейчас» — конкретные команды-глаголы.

```bash
# Создать
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx --replicas=3

# Изменить
kubectl set image deployment/web nginx=nginx:1.25
kubectl scale deployment/web --replicas=5

# Удалить
kubectl delete deployment web
```

**Плюсы:** быстрый для тестов и обучения.
**Минусы:** нет истории, нельзя откатить, невоспроизводимо.

## Декларативный подход

«Я хочу, чтобы было так» — описываем **желаемое состояние** в YAML-файле.

```yaml
# web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
# Одна универсальная команда
kubectl apply -f web-deployment.yaml
```

## Идемпотентность

Ключевое свойство `kubectl apply`:

| Ситуация | Что произойдёт |
|----------|---------------|
| Объект не существует | Создаст |
| Объект существует и совпадает | Ничего не сделает (`unchanged`) |
| Объект существует, но отличается | Обновит diff (только изменённые поля) |

Безопасно запускать `apply` сколько угодно раз — результат одинаковый.

## Анатомия манифеста

Любой ресурс Kubernetes — 4 обязательных поля верхнего уровня:

```yaml
apiVersion: apps/v1          # 1. Версия API (какая схема)
kind: Deployment             # 2. Тип ресурса
metadata:                    # 3. Метаданные (имя, labels, annotations)
  name: web
  namespace: default
  labels:
    app: web
    version: v1
  annotations:
    description: "Production web server"
spec:                        # 4. Спецификация (desired state)
  replicas: 3
  # ...
```

### apiVersion

| API | Ресурсы |
|-----|---------|
| `v1` | Pod, Service, ConfigMap, Secret, PVC, Namespace |
| `apps/v1` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | Ingress, NetworkPolicy |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding |

### Labels vs Annotations

| Свойство | Labels | Annotations |
|----------|--------|------------|
| Назначение | Группировка и выборка | Метаданные для людей/систем |
| Используются в selector | ✅ | ❌ |
| Примеры | `app: web`, `env: prod` | `git-commit: abc123`, `owner: team-backend` |
| Ограничения | 63 символа, alphanumeric | Произвольные строки |

## Генерация YAML (Dry Run)

Не писать YAML с нуля — генерировать шаблон:

```bash
# Deployment
kubectl create deployment web --image=nginx --replicas=3 \
  --dry-run=client -o yaml > web-deployment.yaml

# Service
kubectl expose deployment web --port=80 --type=ClusterIP \
  --dry-run=client -o yaml > web-service.yaml

# Job
kubectl create job backup --image=busybox -- sh -c "echo done" \
  --dry-run=client -o yaml > backup-job.yaml

# Подсмотреть YAML существующего ресурса
kubectl get deployment web -o yaml
```

## Create vs Apply

| | `kubectl create` | `kubectl apply` |
|-|------------------|-----------------|
| Тип | Императивный | Декларативный |
| Логика | «Создай новый» | «Приведи к этому состоянию» |
| Повторный запуск | ❌ `AlreadyExists` | ✅ Обновит или пропустит |
| Production | Нет | Да |
| GitOps | Нет | Да |

## Multi-document YAML

Несколько ресурсов в одном файле через `---`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  # ...
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  # ...
```

```bash
# Применить всё из директории
kubectl apply -f k8s/
kubectl apply -f k8s/ --recursive
```

## GitOps

Естественное продолжение декларативного подхода:

```
Git repo (YAML-манифесты)
  → CI pipeline (lint, validate)
    → CD (kubectl apply / ArgoCD / Flux)
      → Kubernetes cluster
```

- Единственный источник правды — Git
- Все изменения — через Pull Request и code review
- Автоматическая синхронизация кластера с репозиторием
- Полная история: кто, когда, что изменил

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «YAML слишком многословный» | Используй `--dry-run=client -o yaml` для генерации шаблонов. Или Helm/Kustomize |
| «apply и create — одно и то же» | `create` упадёт при повторном запуске. `apply` — идемпотентен |
| «Императивные команды запрещены» | Для debug и одноразовых задач — нормально. Для production — только declarative |
| «Один файл на ресурс — обязательно» | Можно группировать связанные ресурсы через `---`. Но один файл на ресурс — проще для Git |