---
title: "Troubleshooting"
type: how-to
tags: [kubernetes, troubleshoot, debug, crashloop, pending, imagepull, logs, events]
sources:
  docs: "https://kubernetes.io/docs/tasks/debug/"
related:
  - "[[kubernetes/reference/cheatsheet]]"
  - "[[kubernetes/how-to/resource-limits]]"
  - "[[kubernetes/how-to/configure-networking]]"
---

# Troubleshooting Kubernetes

> **TL;DR:** Pod не работает → `kubectl describe pod` + `kubectl logs`.
> `Pending` = scheduling. `CrashLoopBackOff` = приложение падает. `ImagePullBackOff` = образ.
> Всегда проверяйте Events.

## Алгоритм отладки

```
Pod не работает
  │
  ├─ Status: Pending
  │   └─ kubectl describe pod → Events
  │       ├─ Insufficient cpu/memory → увеличить ноду или уменьшить requests
  │       ├─ No nodes match selector → проверить nodeSelector/affinity/taints
  │       └─ PVC Pending → проверить StorageClass и PV
  │
  ├─ Status: ImagePullBackOff
  │   └─ kubectl describe pod → Events
  │       ├─ 404 Not Found → проверить имя образа и тег
  │       ├─ Unauthorized → создать imagePullSecrets
  │       └─ Timeout → проверить сеть ноды
  │
  ├─ Status: CrashLoopBackOff
  │   └─ kubectl logs pod (--previous)
  │       ├─ Ошибка приложения → исправить код или конфигурацию
  │       ├─ OOMKilled → увеличить memory limits
  │       └─ Permission denied → проверить securityContext
  │
  └─ Status: Running, но не отвечает
      ├─ kubectl logs pod → ошибки?
      ├─ kubectl exec -it pod -- curl localhost:PORT → приложение работает?
      ├─ kubectl get endpoints svc → Service видит поды?
      └─ kubectl describe ingress → правила маршрутизации?
```

## Ключевые команды

```bash
# 1. Статус подов
kubectl get pods -o wide
kubectl get pods --field-selector status.phase!=Running

# 2. Events — САМЫЙ ПОЛЕЗНЫЙ источник информации
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events --field-selector involvedObject.name=my-pod

# 3. Описание (events + конфигурация)
kubectl describe pod my-pod
kubectl describe svc my-service
kubectl describe node my-node

# 4. Логи
kubectl logs my-pod
kubectl logs my-pod -c sidecar          # конкретный контейнер
kubectl logs my-pod --previous          # предыдущий (упавший) контейнер
kubectl logs my-pod -f                  # stream
kubectl logs -l app=web --all-containers # все поды по label

# 5. Exec (зайти в контейнер)
kubectl exec -it my-pod -- /bin/sh
kubectl exec -it my-pod -- curl localhost:8080/health
kubectl exec -it my-pod -- env          # проверить env vars
kubectl exec -it my-pod -- cat /etc/resolv.conf  # DNS

# 6. Ресурсы
kubectl top pods --sort-by=memory
kubectl top nodes

# 7. Сеть
kubectl get endpoints my-service        # IP подов за Service
kubectl run debug --image=busybox -it --rm -- wget -qO- http://my-service
```

## Проблемы по статусам

### Pending

```bash
kubectl describe pod my-pod
# Events:
#   Warning  FailedScheduling  Insufficient cpu
#   Warning  FailedScheduling  0/3 nodes are available: 3 node(s) had taint
```

| Причина | Решение |
|---------|---------|
| `Insufficient cpu/memory` | Уменьшить requests или добавить ноды |
| `node(s) had taint` | Добавить toleration или убрать taint с ноды |
| `no persistent volumes available` | Проверить StorageClass, создать PV |
| `Unschedulable` | `kubectl uncordon <node>` |

### ImagePullBackOff

```bash
kubectl describe pod my-pod
# Events:
#   Warning  Failed  Failed to pull image "myapp:v99": not found
```

| Причина | Решение |
|---------|---------|
| Образ не существует | Проверить имя и тег: `docker pull myapp:v99` |
| Private registry | Создать `imagePullSecrets` в Pod spec |
| Rate limit (DockerHub) | Использовать authenticated pull или mirror |

### CrashLoopBackOff

```bash
kubectl logs my-pod --previous
kubectl describe pod my-pod | grep -A5 "Last State"
```

| Причина | Решение |
|---------|---------|
| OOMKilled | Увеличить `limits.memory`. Проверить утечки памяти |
| Exit code 1 | Ошибка приложения — смотреть логи |
| Exit code 137 | Убит сигналом SIGKILL (OOM или preemption) |
| Exit code 0 | Контейнер завершился. Для long-running: не использовать Job |
| Config error | Проверить ConfigMap/Secret смонтированы: `kubectl exec ... -- env` |

### Running, но не отвечает

```bash
# 1. Приложение работает внутри?
kubectl exec -it my-pod -- curl localhost:8080

# 2. Service находит поды?
kubectl get endpoints my-service
# Если пусто — labels не совпадают

# 3. DNS работает?
kubectl run debug --image=busybox -it --rm -- nslookup my-service

# 4. Проверить readinessProbe
kubectl describe pod my-pod | grep -A10 "Readiness"
```

## Debug Pod (временный отладочный контейнер)

```bash
# Запустить отладочный под в том же namespace
kubectl run debug --image=nicolaka/netshoot -it --rm -- /bin/bash

# Внутри:
curl http://my-service:80
nslookup my-service
dig my-service.default.svc.cluster.local
tcpdump -i any port 80

# Ephemeral debug container (K8s 1.23+)
kubectl debug my-pod -it --image=busybox --target=my-container
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Смотрят только `kubectl get pods` | Не видят причину проблемы | `kubectl describe pod` + `kubectl get events` |
| `--previous` не используют | «Логов нет» для CrashLoop | `kubectl logs --previous` показывает логи упавшего контейнера |
| Отладка сети из-за пределов кластера | `curl: connection refused` | Запустить debug pod *внутри* кластера |
| Не проверяют endpoints | «Service не работает» | `kubectl get ep svc-name` — если пусто, labels не совпадают |