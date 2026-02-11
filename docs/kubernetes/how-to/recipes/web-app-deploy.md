---
title: "Рецепт: Деплой веб-приложения"
type: how-to
tags: [kubernetes, recipe, deployment, service, ingress, configmap, secret, hpa]
related:
  - "[[kubernetes/how-to/manage-workloads]]"
  - "[[kubernetes/how-to/configure-networking]]"
  - "[[kubernetes/how-to/resource-limits]]"
---

# Рецепт: Деплой веб-приложения

> Полный набор манифестов: Namespace → ConfigMap → Secret → Deployment → Service → Ingress → HPA.

## Манифесты

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# 01-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  APP_ENV: production
  APP_PORT: "3000"
  LOG_LEVEL: info
---
# 02-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
stringData:
  DATABASE_URL: postgres://user:pass@db:5432/myapp
  JWT_SECRET: super-secret-key-change-me
---
# 03-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: myapp
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0            # zero downtime
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myorg/myapp:v1.0.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # graceful shutdown
---
# 04-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: myapp
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 3000
---
# 05-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: web-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
---
# 06-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Применение

```bash
# Всё сразу
kubectl apply -f manifests/

# Или по файлам
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-configmap.yaml
# ...

# Обновление версии
kubectl set image deploy/web app=myorg/myapp:v1.1.0 -n myapp
kubectl rollout status deploy/web -n myapp

# Откат
kubectl rollout undo deploy/web -n myapp

# Проверка
kubectl get all -n myapp
kubectl get ingress -n myapp
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| readinessProbe не настроен | Трафик идёт на поды до готовности → 502. Всегда настраивать readiness |
| `preStop` hook отсутствует | Connections dropped при rolling update. `sleep 5` даёт время для drain |
| Secret в Git в plain text | Использовать Sealed Secrets или `kubectl create secret` в CI |
| HPA без metrics-server | HPA не масштабирует. Установить metrics-server |