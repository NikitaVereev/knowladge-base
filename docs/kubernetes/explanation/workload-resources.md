---
title: "Workload-ресурсы"
type: explanation
tags: [kubernetes, pod, deployment, statefulset, daemonset, job, cronjob, replicaset, hpa]
sources:
  docs: "https://kubernetes.io/docs/concepts/workloads/"
related:
  - "[[kubernetes/explanation/architecture]]"
  - "[[kubernetes/explanation/networking]]"
  - "[[kubernetes/how-to/manage-workloads]]"
---

# Workload-ресурсы

> **TL;DR:** Pod — минимальная единица. Deployment — для stateless (веб, API).
> StatefulSet — для stateful (БД). DaemonSet — по одному на ноду. Job/CronJob — разовые/периодические задачи.
> Никогда не создавайте Pod напрямую — используйте контроллеры.

## Pod

Минимальная единица деплоя. Группа из одного или нескольких контейнеров с общей сетью и storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
```

**Правило:** Никогда не создавайте Pod напрямую в production. Используйте Deployment/StatefulSet — они обеспечивают self-healing и scaling.

### Probes (проверки здоровья)

| Probe | Вопрос | Действие при failure |
|-------|--------|---------------------|
| **livenessProbe** | «Процесс жив?» | Restart контейнера |
| **readinessProbe** | «Готов принимать трафик?» | Убрать из Service endpoints |
| **startupProbe** | «Запустился?» | Ждать до timeout, потом restart |

Типы проверок: `httpGet`, `tcpSocket`, `exec`, `grpc`.

## Deployment

Управляет stateless-приложениями. Обеспечивает: указанное количество реплик, rolling update, rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # на 1 под больше во время обновления
      maxUnavailable: 0        # 0 недоступных = zero-downtime
  template:                    # ← Pod Template
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myapp:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
```

### Иерархия

```
Deployment → ReplicaSet → Pod
                          Pod
                          Pod
```

Deployment создаёт ReplicaSet, который поддерживает нужное количество подов. При обновлении image — создаётся новый ReplicaSet, старый плавно сворачивается.

### Update и Rollback

```bash
# Обновить image
kubectl set image deployment/web app=myapp:v1.3.0

# Следить за прогрессом
kubectl rollout status deployment/web

# История версий
kubectl rollout history deployment/web

# Откат к предыдущей версии
kubectl rollout undo deployment/web

# Откат к конкретной ревизии
kubectl rollout undo deployment/web --to-revision=3

# Рестарт подов (пересоздание)
kubectl rollout restart deployment/web
```

## StatefulSet

Для stateful-приложений (БД, Kafka, Elasticsearch), где важны: стабильные сетевые идентификаторы, порядок запуска, персональные PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres        # ← обязательно, Headless Service
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
  volumeClaimTemplates:        # ← каждой реплике свой PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

### Отличия от Deployment

| Свойство | Deployment | StatefulSet |
|----------|-----------|-------------|
| Имена подов | Случайные (`web-7d5bc6-x4k2z`) | Порядковые (`postgres-0`, `postgres-1`) |
| Порядок запуска | Параллельно | Последовательно (0 → 1 → 2) |
| PVC | Общий (или без) | Персональный для каждого пода |
| DNS | Через Service | `<pod>.<service>.<ns>.svc.cluster.local` |
| Удаление | PVC удаляются | PVC **сохраняются** |
| Обновление | Параллельно | По одному (reverse order: 2 → 1 → 0) |

## DaemonSet

Запускает **ровно один под на каждой ноде**. При добавлении новой ноды — под создаётся автоматически.

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
              hostPort: 9100
```

Когда использовать: мониторинг (node-exporter, Datadog), логирование (Fluentd, Filebeat), сетевые плагины (Calico, Cilium).

## Job

Запускает задачу до успешного завершения. Под не перезапускается после `exit 0`.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3              # максимум 3 попытки при ошибке
  activeDeadlineSeconds: 300   # timeout 5 минут
  template:
    spec:
      restartPolicy: Never     # обязательно Never или OnFailure
      containers:
        - name: migrate
          image: myapp:v1.2.0
          command: ["python", "manage.py", "migrate"]
```

### Параллельный Job

```yaml
spec:
  completions: 10              # нужно 10 успешных завершений
  parallelism: 3               # запускать по 3 параллельно
```

## CronJob

Job по расписанию (cron-синтаксис):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 3 * * *"       # каждый день в 03:00
  concurrencyPolicy: Forbid    # не запускать новый, если предыдущий ещё работает
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16
              command:
                - sh
                - -c
                - pg_dumpall -h postgres -U admin | gzip > /backups/$(date +%Y%m%d).sql.gz
```

## HPA (Horizontal Pod Autoscaler)

Автоматическое масштабирование по метрикам:

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
          averageUtilization: 70       # scale up при >70% CPU
```

Требуется `metrics-server` в кластере.

## Какой ресурс выбрать?

| Задача | Ресурс |
|--------|--------|
| Веб-сервер, API, frontend | **Deployment** |
| БД, Kafka, Elasticsearch | **StatefulSet** |
| Мониторинг-агент, лог-коллектор | **DaemonSet** |
| Миграция БД, обработка данных | **Job** |
| Бэкап по расписанию, отчёты | **CronJob** |

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «Pod = Deployment» | Pod — одноразовый. Deployment — контроллер, пересоздающий поды |
| «StatefulSet нужен для всего с БД» | Если БД — managed (RDS, CloudSQL) — Deployment достаточно для приложения |
| «Нет requests/limits = ок» | Без requests Scheduler не знает, куда поставить. Без limits — один под может убить ноду (OOMKill) |
| «restartPolicy: Always для Job» | Job требует `Never` или `OnFailure`. `Always` — для Deployment |
| «HPA без readinessProbe» | Трафик пойдёт на под до его готовности → 502 ошибки |
