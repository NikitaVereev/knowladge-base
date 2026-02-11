---
title: "03 — Stateful-приложение"
type: tutorial
tags: [kubernetes, tutorial, configmap, secret, pvc, statefulset, env, volumes]
sources:
  docs: "https://kubernetes.io/docs/tutorials/stateful-application/"
related:
  - "[[kubernetes/tutorials/02-first-deployment]]"
  - "[[kubernetes/tutorials/04-ingress-and-tls]]"
  - "[[kubernetes/explanation/storage]]"
  - "[[kubernetes/how-to/manage-storage]]"
---

# Tutorial 03 — Stateful-приложение

> **Цель:** Развернуть приложение с БД: ConfigMap для конфигурации, Secret для паролей,
> PVC для данных PostgreSQL. Понять связь ConfigMap → Pod, Secret → Pod, PVC → StatefulSet.

**Время:** ~40 минут
**Требования:** Пройден Tutorial 02. Minikube запущен.

## Шаг 1. Namespace

```bash
kubectl create namespace tutorial
kubectl config set-context --current --namespace=tutorial
```

## Шаг 2. ConfigMap (конфигурация)

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: tutorial
data:
  APP_ENV: production
  APP_PORT: "3000"
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: myapp
```

```bash
kubectl apply -f configmap.yaml

# Проверить
kubectl get cm app-config
kubectl describe cm app-config
```

## Шаг 3. Secret (секреты)

```bash
# Императивно (не попадёт в Git)
kubectl create secret generic db-secret \
  --from-literal=POSTGRES_USER=myapp \
  --from-literal=POSTGRES_PASSWORD=mysecretpass \
  -n tutorial

# Проверить (значения закодированы в base64)
kubectl get secret db-secret -o yaml
# echo "bXlzZWNyZXRwYXNz" | base64 -d  → mysecretpass
```

## Шаг 4. PostgreSQL (StatefulSet + PVC)

```yaml
# postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: tutorial
spec:
  clusterIP: None                    # Headless — обязателен для StatefulSet
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: tutorial
spec:
  serviceName: postgres
  replicas: 1
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
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: db-secret
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "myapp"]
            initialDelaySeconds: 5
            periodSeconds: 5
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f postgres.yaml

# Подождать запуска
kubectl get pods -w
# postgres-0   1/1     Running   0   30s

# Проверить PVC (создан автоматически)
kubectl get pvc
# NAME              STATUS   VOLUME    CAPACITY   ACCESS MODES
# data-postgres-0   Bound    pvc-xxx   1Gi        RWO

# Подключиться к PostgreSQL
kubectl exec -it postgres-0 -- psql -U myapp
# \l   — список баз
# \q   — выйти
```

## Шаг 5. Приложение (Deployment)

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: tutorial
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: nginx:alpine
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: POSTGRES_PASSWORD
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: tutorial
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f app.yaml

# Проверить что env vars доставлены
kubectl exec deploy/web -- env | grep -E "APP_|DB_"
# APP_ENV=production
# APP_PORT=3000
# DB_HOST=postgres
# DB_USER=myapp
# DB_PASSWORD=mysecretpass

# Проверить DNS (приложение → БД)
kubectl exec deploy/web -- nslookup postgres
# Address: 10.244.0.5

# Проверить доступ к приложению
minikube service web-service -n tutorial --url
```

## Шаг 6. Проверка персистентности

```bash
# Создать таблицу в PostgreSQL
kubectl exec -it postgres-0 -- psql -U myapp -c "CREATE TABLE test (id serial, msg text);"
kubectl exec -it postgres-0 -- psql -U myapp -c "INSERT INTO test (msg) VALUES ('persistent data');"

# Удалить pod (StatefulSet пересоздаст его)
kubectl delete pod postgres-0

# Подождать пересоздания
kubectl get pods -w

# Данные сохранились!
kubectl exec -it postgres-0 -- psql -U myapp -c "SELECT * FROM test;"
# id |      msg
# ----+----------------
#   1 | persistent data
```

## Шаг 7. Очистка

```bash
kubectl delete namespace tutorial
# Удалит ВСЁ внутри namespace, включая PVC
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| ConfigMap | Конфигурация приложения через `envFrom` и `valueFrom` |
| Secret | Пароли БД, base64 кодирование, `secretRef` |
| StatefulSet | Стабильное имя `postgres-0`, упорядоченный запуск |
| Headless Service | `clusterIP: None` — обязателен для StatefulSet |
| PVC | `volumeClaimTemplates` — каждый под получает свой диск |
| Персистентность | Pod удалён → PVC сохраняет данные → новый Pod подхватывает |
| subPath | `subPath: pgdata` — избегает `lost+found` в корне тома |

## Что дальше

→ [[kubernetes/tutorials/04-ingress-and-tls]] — Ingress для HTTP-маршрутизации и TLS

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Secret в YAML в Git | Утечка паролей | `kubectl create secret` императивно или Sealed Secrets |
| Забыли `subPath` для Postgres | `initdb: directory not empty` | `subPath: pgdata` в volumeMounts |
| ConfigMap обновлён, под не обновился | Старые env vars | `kubectl rollout restart deploy/web` |
| StatefulSet без Headless Service | Ошибка создания | Service с `clusterIP: None` обязателен |
| `kubectl delete namespace` удалил PVC | Данные потеряны | PVC в namespace удаляются вместе с ним. Backup! |
