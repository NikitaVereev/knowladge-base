---
title: "Управление доступом (RBAC)"
type: how-to
tags: [kubernetes, rbac, role, clusterrole, serviceaccount, binding, security]
sources:
  docs: "https://kubernetes.io/docs/reference/access-authn-authz/rbac/"
related:
  - "[[kubernetes/reference/resource-reference]]"
  - "[[kubernetes/how-to/manage-workloads]]"
---

# Управление доступом (RBAC)

> **TL;DR:** Role = разрешения в namespace. ClusterRole = на весь кластер.
> ServiceAccount = идентификация подов. RoleBinding связывает Role + SA/User.

## Модель RBAC

```
Кто?              Что может?           Где?
─────             ──────────           ────
User              Role                 Namespace
ServiceAccount    ClusterRole          Cluster-wide
Group
        └──── RoleBinding / ClusterRoleBinding ────┘
```

## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
```

Привязка к поду:

```yaml
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: myapp:v1
```

```bash
# Создать
kubectl create serviceaccount app-sa

# Посмотреть
kubectl get sa
```

## Role (namespace-level)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]              # "" = core API (pods, services, configmaps)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  - apiGroups: ["apps"]          # apps API (deployments, statefulsets)
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### Verbs

| Verb | HTTP | Описание |
|------|------|----------|
| `get` | GET | Получить один ресурс |
| `list` | GET | Получить список |
| `watch` | GET (stream) | Подписка на изменения |
| `create` | POST | Создать |
| `update` | PUT | Полное обновление |
| `patch` | PATCH | Частичное обновление |
| `delete` | DELETE | Удалить |
| `*` | all | Все операции |

## RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-pod-reader
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## ClusterRole + ClusterRoleBinding

Для ресурсов без namespace (nodes, PV) или доступа ко всем namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "pods", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-binding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

## Быстрые команды

```bash
# Проверить права (can-i)
kubectl auth can-i list pods --as=system:serviceaccount:default:app-sa
kubectl auth can-i create deployments
kubectl auth can-i '*' '*'             # admin?

# Создать role + binding императивно
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding app-reader --role=pod-reader --serviceaccount=default:app-sa

# Cluster-level
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding node-reader-binding --clusterrole=node-reader --serviceaccount=monitoring:prometheus-sa
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Pod без ServiceAccount | Использует `default` SA с минимальными правами | Создать SA + Role + Binding |
| Role вместо ClusterRole для nodes | `Forbidden: nodes is cluster-scoped` | Nodes, PV, Namespaces — только ClusterRole |
| `apiGroups: [""]` забыт | Role не работает для core ресурсов (pods, services) | `""` = core API group |
| Слишком широкие права (`*`) | Уязвимость: под может делать что угодно | Principle of least privilege: только нужные verbs и resources |