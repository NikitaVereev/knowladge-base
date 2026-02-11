---
title: "Kubernetes"
type: index
tags: [kubernetes, k8s, orchestration, containers, cloud-native]
---

# Kubernetes

Оркестрация контейнеров: автоматический деплой, масштабирование, self-healing. Декларативная модель (YAML-манифесты), reconciliation loop (desired state → actual state).

## Explanation (Концепции)

| Документ | Описание |
|----------|----------|
| [[kubernetes/explanation/architecture]] | Control Plane, Worker Nodes, API Server, etcd, Scheduler, Kubelet |
| [[kubernetes/explanation/manifests-philosophy]] | Императивный vs декларативный, `kubectl apply`, идемпотентность |
| [[kubernetes/explanation/workload-resources]] | Pod, Deployment, StatefulSet, DaemonSet, Job, CronJob |
| [[kubernetes/explanation/networking]] | Service типы (ClusterIP, NodePort, LB), Headless, Ingress, DNS |
| [[kubernetes/explanation/storage]] | PV, PVC, StorageClass, Dynamic Provisioning, Access Modes, Reclaim Policy |

## How-to (Практические руководства)

| Документ | Описание |
|----------|----------|
| [[kubernetes/how-to/manage-workloads]] | Deployment, StatefulSet, DaemonSet, Job, scaling, rolling update |
| [[kubernetes/how-to/configure-networking]] | Service, Ingress, NetworkPolicy, DNS, port-forward |
| [[kubernetes/how-to/manage-storage]] | ConfigMap, Secret, PVC, StorageClass, emptyDir |
| [[kubernetes/how-to/rbac]] | Role, ClusterRole, ServiceAccount, RoleBinding, can-i |
| [[kubernetes/how-to/resource-limits]] | requests/limits, QoS, LimitRange, ResourceQuota, HPA |
| [[kubernetes/how-to/troubleshoot]] | Алгоритм отладки, Pending/CrashLoop/ImagePull, debug pod |
| [[kubernetes/how-to/helm-basics]] | Helm install/upgrade/rollback, values.yaml, создание чарта |

## Recipes (Готовые решения)(В процессе)

| Рецепт | Описание |
|--------|----------|
| [[kubernetes/how-to/recipes/web-app-deploy]] | Полный набор: Deployment + Service + Ingress + HPA + probes |

## Reference (Справочники)

| Документ | Описание |
|----------|----------|
| [[kubernetes/reference/cheatsheet]] | kubectl: get, describe, logs, exec, apply, rollout, port-forward |
| [[kubernetes/reference/yaml-templates]] | Шаблоны манифестов: Pod, Deployment, Service, Ingress, PVC |
| [[kubernetes/reference/resource-reference]] | Все ресурсы: Kind, Short, apiVersion, NS, описание |

## Tutorials (Пошаговые уроки)

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[kubernetes/tutorials/01-local-setup]] | kubectl + Minikube, запуск кластера, k9s |
| 02 | [[kubernetes/tutorials/02-first-deployment]] | Deployment, Service, масштабирование, rolling update, rollback |
| 03 | [[kubernetes/tutorials/03-stateful-app]] | ConfigMap, Secret, PVC, StatefulSet (PostgreSQL) |
| 04 | [[kubernetes/tutorials/04-ingress-and-tls]] | Ingress маршрутизация, TLS, cert-manager |

## Быстрый старт

```bash
# 1. Установка
brew install kubectl minikube    # или pacman -S kubectl minikube

# 2. Запуск кластера
minikube start --driver=docker

# 3. Первый деплой
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80 --type=NodePort
minikube service web --url

# 4. Декларативный подход
kubectl apply -f deployment.yaml
kubectl rollout status deploy/web
```

## Связанные разделы

- [[docker/index]] — Docker (контейнеры, на которых работает K8s)
- [[ansible/index]] — Ansible (автоматизация настройки серверов)
