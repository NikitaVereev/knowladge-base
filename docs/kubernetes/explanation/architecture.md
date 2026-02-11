---
title: "Архитектура Kubernetes"
type: explanation
tags: [kubernetes, architecture, control-plane, worker-node, api-server, etcd, kubelet, scheduler]
sources:
  docs: "https://kubernetes.io/docs/concepts/architecture/"
related:
  - "[[kubernetes/explanation/workload-resources]]"
  - "[[kubernetes/explanation/manifests-philosophy]]"
  - "[[kubernetes/tutorials/01-local-setup]]"
---

# Архитектура Kubernetes

> **TL;DR:** Кластер = Control Plane (мозг) + Worker Nodes (мышцы).
> API Server — единственная точка входа. Etcd — source of truth.
> Scheduler выбирает ноду, Kubelet запускает контейнеры.

Kubernetes — распределённая система из множества компонентов, работающих согласованно. Кластер делится на две части: **Control Plane** (управление) и **Worker Nodes** (выполнение).

## Control Plane (Управляющий слой)

Компоненты, принимающие решения. В production дублируются для отказоустойчивости (обычно 3 или 5 экземпляров).

### API Server (`kube-apiserver`)

Центральная точка входа. Единственный компонент, с которым вы взаимодействуете через `kubectl`.

- Принимает REST-запросы, проверяет авторизацию (RBAC), валидирует данные
- Единственный, кто имеет право писать в Etcd
- Все остальные компоненты общаются **только через API Server**

### Etcd

Распределённое key-value хранилище. Хранит **всё** состояние кластера: какие поды запущены, конфиги, секреты, RBAC-правила.

- Source of truth — если потеряете Etcd, потеряете кластер
- Raft-консенсус — для отказоустойчивости нужно нечётное число узлов (3, 5)
- Бэкап Etcd — критически важная операция в production

### Scheduler (`kube-scheduler`)

Планировщик — решает, **где** запустить новый под.

- Следит за подами без назначенной ноды (`nodeName: ""`)
- Учитывает: requests/limits ресурсов, affinity/anti-affinity, taints/tolerations, topology spread
- Не запускает поды — только назначает ноду

### Controller Manager (`kube-controller-manager`)

Набор контроллеров, приводящих текущее состояние к желаемому (reconciliation loop).

| Контроллер | Что делает |
|-----------|-----------|
| ReplicaSet | Поддерживает нужное количество реплик |
| Deployment | Управляет rolling update и rollback |
| Node | Следит за здоровьем нод |
| Job | Запускает задачи до успешного завершения |
| ServiceAccount | Создаёт дефолтные SA для namespace |

## Worker Nodes (Рабочие узлы)

Машины, на которых запускаются ваши приложения.

### Kubelet

Агент на каждой ноде.

- Получает от API Server список подов для запуска
- Управляет Container Runtime (containerd, CRI-O)
- Выполняет health-проверки (liveness, readiness, startup probes)
- Рапортует статус обратно в API Server

### Kube-proxy

Сетевой компонент на каждой ноде.

- Реализует абстракцию Service (iptables/IPVS правила)
- Обеспечивает балансировку трафика между подами
- Поддерживает ClusterIP, NodePort, LoadBalancer

### Container Runtime

Непосредственный запуск контейнеров.

- **containerd** — стандарт де-факто (с K8s 1.24)
- **CRI-O** — минималистичная альтернатива
- Docker Engine — не поддерживается напрямую с 1.24 (только через containerd)

## Как это работает вместе

Когда вы выполняете `kubectl apply -f deployment.yaml`:

```
1. kubectl        → HTTP POST → API Server
2. API Server     → валидация + RBAC → сохранение в Etcd
3. Scheduler      → watch: новый под без ноды → назначает node-1
4. Kubelet (node-1) → watch: назначен новый под → Container Runtime
5. Container Runtime → pull image → start container
6. Kubelet        → рапортует статус → API Server → Etcd
```

```
┌─────────────────── Control Plane ───────────────────┐
│                                                       │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │  Etcd    │  │ Scheduler│  │  Controller Manager  │ │
│  └────┬─────┘  └────┬─────┘  └──────────┬──────────┘ │
│       │              │                   │             │
│       └──────────────┼───────────────────┘             │
│                      │                                 │
│              ┌───────┴────────┐                        │
│              │   API Server   │ ← kubectl / CI/CD      │
│              └───────┬────────┘                        │
└──────────────────────┼────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
   ┌──────┴──────┐ ┌──┴──────┐ ┌──┴──────┐
   │  Worker 1   │ │ Worker 2│ │ Worker 3│
   │  ┌───────┐  │ │         │ │         │
   │  │Kubelet│  │ │         │ │         │
   │  │Kube-  │  │ │  ...    │ │  ...    │
   │  │proxy  │  │ │         │ │         │
   │  │Runtime│  │ │         │ │         │
   │  └───────┘  │ │         │ │         │
   └─────────────┘ └─────────┘ └─────────┘
```

## Kubernetes vs Docker Compose

| Свойство | Docker Compose | Kubernetes |
|----------|---------------|-----------|
| Масштаб | 1 сервер | Кластер (1-1000+ нод) |
| Self-healing | `restart: always` | Scheduler перемещает под на другую ноду |
| Scaling | Вручную | `kubectl scale`, HPA (автоматически) |
| Networking | bridge network | Overlay network (CNI), Service Discovery |
| Storage | bind mounts, volumes | PV/PVC, StorageClass, dynamic provisioning |
| Rolling update | `docker compose up -d` | Deployment strategy, zero-downtime |
| Secrets | `.env` файлы | Encrypted Secrets, RBAC |

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «K8s нужен для любого проекта» | Для 1-3 серверов Docker Compose проще и дешевле. K8s оправдан при масштабировании и высокой доступности |
| «Pod = контейнер» | Pod — группа контейнеров с общей сетью и storage. Обычно 1 контейнер, но бывают sidecar-паттерны |
| «Etcd можно не бэкапить» | Потеря Etcd = потеря кластера. `etcdctl snapshot save` — обязательная процедура |
| «kubectl = единственный клиент» | API Server принимает любые HTTP-запросы. CI/CD, Terraform, Ansible — все работают через REST API |
