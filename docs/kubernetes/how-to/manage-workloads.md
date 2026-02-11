---
title: "Управление нагрузками (Workloads)"
type: how-to
tags: [kubernetes, deployment, statefulset, daemonset, job, cronjob, rolling-update, scaling]
sources:
  docs: "https://kubernetes.io/docs/concepts/workloads/"
related:
  - "[[kubernetes/explanation/workload-resources]]"
  - "[[kubernetes/how-to/resource-limits]]"
  - "[[kubernetes/reference/yaml-templates]]"
---

# Управление нагрузками (Workloads)

> **TL;DR:** Deployment — stateless, StatefulSet — stateful, DaemonSet — по поду на ноду.
> Job — разовая задача, CronJob — по расписанию. `kubectl rollout` — управление обновлениями.

## Deployment (Stateless)

### Создание и обновление

```bash
# Императивно (для тестов)
kubectl create deployment web --image=nginx:1.24 --replicas=3

# Декларативно (production)
kubectl apply -f deployment.yaml

# Обновить образ
kubectl set image deploy/web nginx=nginx:1.25

# Наблюдать за выкаткой
kubectl rollout status deploy/web

# История версий
kubectl rollout history deploy/web

# Откат
kubectl rollout undo deploy/web
kubectl rollout undo deploy/web --to-revision=2
```

### Стратегии обновления

```yaml
spec:
  strategy:
    type: RollingUpdate            # или Recreate
    rollingUpdate:
      maxSurge: 1                  # сколько СВЕРХ replicas допустимо
      maxUnavailable: 0            # сколько недоступных допустимо (0 = zero downtime)
```

- `RollingUpdate` (default): постепенная замена подов. Zero downtime.
- `Recreate`: убить все старые, запустить все новые. Используется если старая и новая версии несовместимы.

### Масштабирование

```bash
# Вручную
kubectl scale deploy/web --replicas=5

# Автоматически (HPA)
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=70
```

HPA в манифесте:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## StatefulSet (Stateful)

Для приложений с состоянием: стабильные имена (`web-0`, `web-1`), упорядоченный запуск/остановка, персональные PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres            # обязательно — Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:            # каждый под получит свой PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

```bash
# Pods: postgres-0, postgres-1, postgres-2 (стабильные имена)
# PVCs: data-postgres-0, data-postgres-1, data-postgres-2

# DNS: postgres-0.postgres.default.svc.cluster.local
```

## DaemonSet (По поду на ноду)

Логи, мониторинг, сетевые агенты — один под на каждой ноде.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
              hostPort: 9100       # привязка к порту ноды
```

## Job и CronJob

```yaml
# Разовая задача
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3                  # кол-во попыток
  activeDeadlineSeconds: 300       # таймаут
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:v1
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
---
# Периодическая задача
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 3 * * *"           # каждый день в 3:00
  concurrencyPolicy: Forbid        # не запускать если предыдущий ещё работает
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16
              command: ["pg_dumpall", "-U", "postgres"]
          restartPolicy: OnFailure
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Deployment для БД | Данные теряются при пересоздании пода | Использовать StatefulSet + PVC |
| `maxUnavailable: 25%` при 2 репликах | 1 под недоступен = 50% downtime | `maxUnavailable: 0` для малого числа реплик |
| StatefulSet без Headless Service | Ошибка валидации | Создать Service с `clusterIP: None` |
| Job без `backoffLimit` | Бесконечные retry при ошибке | Указать `backoffLimit: 3` и `activeDeadlineSeconds` |
| CronJob без `concurrencyPolicy` | Наложение задач если предыдущая не завершилась | `concurrencyPolicy: Forbid` |