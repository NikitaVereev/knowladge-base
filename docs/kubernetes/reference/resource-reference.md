---
title: "Справочник ресурсов Kubernetes"
type: reference
tags: [kubernetes, resources, aliases, api-versions, shortnames, namespaced]
sources:
  docs: "https://kubernetes.io/docs/reference/kubectl/"
related:
  - "[[kubernetes/reference/cheatsheet]]"
  - "[[kubernetes/reference/yaml-templates]]"
  - "[[kubernetes/explanation/workload-resources]]"
---

# Справочник ресурсов Kubernetes

> **Справочник:** Все ресурсы K8s: сокращения, API-версии, namespaced или нет.
> `kubectl api-resources` — актуальный список для вашего кластера.

## Workloads (Нагрузки)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| Pod | `po` | v1 | ✓ | Минимальная единица — один или несколько контейнеров |
| Deployment | `deploy` | apps/v1 | ✓ | Управление stateless-приложениями, rolling updates |
| ReplicaSet | `rs` | apps/v1 | ✓ | Поддерживает N реплик подов (управляется Deployment) |
| StatefulSet | `sts` | apps/v1 | ✓ | Stateful-приложения: стабильные имена, порядок, PVC |
| DaemonSet | `ds` | apps/v1 | ✓ | По одному поду на каждой ноде (мониторинг, логи) |
| Job | `job` | batch/v1 | ✓ | Задача, выполняемая до завершения |
| CronJob | `cj` | batch/v1 | ✓ | Периодическая задача по расписанию |
| ReplicationController | `rc` | v1 | ✓ | Устаревший предшественник ReplicaSet |

## Networking (Сеть)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| Service | `svc` | v1 | ✓ | Стабильный endpoint для подов (ClusterIP/NodePort/LB) |
| Endpoints | `ep` | v1 | ✓ | Список IP-адресов подов за Service |
| Ingress | `ing` | networking.k8s.io/v1 | ✓ | HTTP/HTTPS маршрутизация по доменам |
| IngressClass | — | networking.k8s.io/v1 | ✗ | Класс Ingress-контроллера (nginx, traefik) |
| NetworkPolicy | `netpol` | networking.k8s.io/v1 | ✓ | Правила сетевого доступа между подами |

## Configuration (Конфигурация)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| ConfigMap | `cm` | v1 | ✓ | Конфигурация в key-value (не секретная) |
| Secret | — | v1 | ✓ | Секретные данные (base64, не шифрованные по умолчанию!) |
| ServiceAccount | `sa` | v1 | ✓ | Идентификация подов для RBAC |

## Storage (Хранение)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| PersistentVolume | `pv` | v1 | ✗ | Физический том (создаётся админом или динамически) |
| PersistentVolumeClaim | `pvc` | v1 | ✓ | Запрос на том (создаётся разработчиком) |
| StorageClass | `sc` | storage.k8s.io/v1 | ✗ | Класс хранилища (SSD, HDD, provisioner) |

## Cluster (Кластер)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| Node | `no` | v1 | ✗ | Физическая/виртуальная машина кластера |
| Namespace | `ns` | v1 | ✗ | Логическое разделение ресурсов |
| Event | `ev` | v1 | ✓ | Системные события (создание, ошибки) |

## RBAC (Права доступа)

| Kind | Short | apiVersion | NS | Описание |
|------|-------|------------|-----|----------|
| Role | — | rbac.authorization.k8s.io/v1 | ✓ | Набор разрешений в namespace |
| ClusterRole | — | rbac.authorization.k8s.io/v1 | ✗ | Набор разрешений на уровне кластера |
| RoleBinding | — | rbac.authorization.k8s.io/v1 | ✓ | Привязка Role к пользователю/SA |
| ClusterRoleBinding | — | rbac.authorization.k8s.io/v1 | ✗ | Привязка ClusterRole к пользователю/SA |

## Полезные команды

```bash
# Полный список ресурсов вашего кластера
kubectl api-resources

# Только namespaced ресурсы
kubectl api-resources --namespaced=true

# Ресурсы конкретной API-группы
kubectl api-resources --api-group=apps

# Версии API
kubectl api-versions

# Документация по ресурсу
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy
kubectl explain service.spec --recursive
```

## apiVersion — когда какой?

| apiVersion | Ресурсы |
|-----------|---------|
| `v1` | Pod, Service, ConfigMap, Secret, PV, PVC, Namespace, Node, SA |
| `apps/v1` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | Ingress, IngressClass, NetworkPolicy |
| `storage.k8s.io/v1` | StorageClass |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `autoscaling/v2` | HorizontalPodAutoscaler |
| `policy/v1` | PodDisruptionBudget |
