---
title: "02 — Первый Deployment"
type: tutorial
tags: [kubernetes, tutorial, deployment, service, pod, labels, replicas, port-forward]
sources:
  docs: "https://kubernetes.io/docs/tutorials/kubernetes-basics/"
related:
  - "[[kubernetes/tutorials/01-local-setup]]"
  - "[[kubernetes/tutorials/03-stateful-app]]"
  - "[[kubernetes/explanation/workload-resources]]"
  - "[[kubernetes/explanation/manifests-philosophy]]"
---

# Tutorial 02 — Первый Deployment

> **Цель:** Создать Deployment (Nginx), масштабировать, обновить, откатить.
> Создать Service для доступа. Понять lifecycle: Deployment → ReplicaSet → Pod.

**Время:** ~30 минут
**Требования:** Пройден Tutorial 01. Minikube запущен.

## Шаг 1. Императивный старт (быстрая проверка)

```bash
# Создать Deployment
kubectl create deployment web --image=nginx:1.24 --replicas=2

# Проверить
kubectl get deployments
kubectl get pods
kubectl get replicasets

# Удалить (мы пересоздадим декларативно)
kubectl delete deployment web
```

## Шаг 2. Декларативный манифест

Создайте файл `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web                   # ← должен совпадать с template.labels
  template:
    metadata:
      labels:
        app: web                 # ← метка подов
    spec:
      containers:
        - name: nginx
          image: nginx:1.24
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "250m"
```

Применяем:

```bash
kubectl apply -f deployment.yaml
```

## Шаг 3. Исследование

```bash
# Deployment
kubectl get deploy web
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# web    2/2     2            2           30s

# ReplicaSet (создан автоматически)
kubectl get rs
# NAME             DESIRED   CURRENT   READY
# web-7d4b8c6f5    2         2         2

# Pods (имена = deployment-replicaset-random)
kubectl get pods -o wide
# NAME                   READY   STATUS    IP           NODE
# web-7d4b8c6f5-abc12   1/1     Running   10.244.0.5   minikube
# web-7d4b8c6f5-def34   1/1     Running   10.244.0.6   minikube

# Подробная информация
kubectl describe deploy web

# Labels — ключ к пониманию K8s
kubectl get pods --show-labels
kubectl get pods -l app=web
```

## Шаг 4. Доступ к приложению (Service)

Поды имеют эфемерные IP. Service — стабильная точка доступа.

Создайте `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort                   # для доступа снаружи Minikube
  selector:
    app: web                       # ← ищет поды с этой меткой
  ports:
    - protocol: TCP
      port: 80                     # порт Service
      targetPort: 80               # порт контейнера
      nodePort: 30080              # порт на ноде (30000-32767)
```

```bash
kubectl apply -f service.yaml

# Проверить
kubectl get svc web-service
# NAME          TYPE       CLUSTER-IP     PORT(S)        AGE
# web-service   NodePort   10.96.45.123   80:30080/TCP   5s

# Доступ через Minikube
minikube service web-service --url
# http://192.168.49.2:30080

# Или через port-forward (работает с любым типом Service)
kubectl port-forward svc/web-service 8080:80
# Откройте http://localhost:8080
```

## Шаг 5. Масштабирование

```bash
# Императивно
kubectl scale deploy web --replicas=5

# Проверить
kubectl get pods
# 5 подов

# Декларативно (изменить replicas в deployment.yaml)
# replicas: 3
kubectl apply -f deployment.yaml
kubectl get pods
# 3 пода (2 лишних удалены)
```

## Шаг 6. Обновление (Rolling Update)

Изменяем образ с `nginx:1.24` на `nginx:1.25`:

```bash
# Вариант 1: Императивно
kubectl set image deploy/web nginx=nginx:1.25

# Вариант 2: Декларативно (изменить image в deployment.yaml)
kubectl apply -f deployment.yaml

# Наблюдать за выкаткой
kubectl rollout status deploy web
# Waiting for deployment "web" rollout to finish: 1 out of 3 new replicas updated...
# deployment "web" successfully rolled out

# Что происходит внутри
kubectl get rs
# NAME             DESIRED   CURRENT   READY
# web-7d4b8c6f5    0         0         0        ← старый RS (масштабирован в 0)
# web-8e5c9d7f6    3         3         3        ← новый RS

# История ревизий
kubectl rollout history deploy web
```

## Шаг 7. Откат (Rollback)

```bash
# Откатиться к предыдущей версии
kubectl rollout undo deploy web

# Откатиться к конкретной ревизии
kubectl rollout history deploy web             # посмотреть номера
kubectl rollout undo deploy web --to-revision=1

# Проверить
kubectl describe deploy web | grep Image
```

## Шаг 8. Очистка

```bash
# Удалить всё что создали
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml

# Или удалить всё с определённой меткой
kubectl delete all -l app=web
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| Deployment | Управляет ReplicaSet, который управляет Pods |
| Labels + Selector | `app: web` связывает Deployment → Pods → Service |
| Replicas | `kubectl scale` или `replicas:` в YAML |
| Rolling Update | Новый RS поднимается, старый масштабируется в 0 |
| Rollback | `kubectl rollout undo` — мгновенный откат |
| Service (NodePort) | Стабильный endpoint + доступ снаружи через порт ноды |
| port-forward | Быстрый доступ без Service для отладки |
| `kubectl apply` | Декларативный подход — идемпотентный |

## Иерархия объектов

```
Deployment (web)
  └── ReplicaSet (web-8e5c9d7f6)       # создаётся автоматически
        ├── Pod (web-8e5c9d7f6-abc12)
        ├── Pod (web-8e5c9d7f6-def34)
        └── Pod (web-8e5c9d7f6-ghi56)

Service (web-service)
  └── selector: app=web  ──────────────→ находит все 3 Pod'а
```

## Что дальше

→ [[kubernetes/tutorials/03-stateful-app]] — ConfigMap, Secret, PVC, StatefulSet

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| selector не совпадает с template labels | `selector does not match template` | `spec.selector.matchLabels` == `spec.template.metadata.labels` |
| Нет resource requests/limits | Pod убит OOMKiller или не ходит по нодам | Всегда указывать `resources.requests` и `resources.limits` |
| `ImagePullBackOff` | Ошибка в имени образа или нет доступа | `kubectl describe pod <name>` → Events |
| Service не находит поды | `kubectl get endpoints` — пусто | Проверить selector Service == labels Pod |
| NodePort 80 вместо 30000+ | `invalid port: must be 30000-32767` | NodePort в диапазоне 30000-32767 |
