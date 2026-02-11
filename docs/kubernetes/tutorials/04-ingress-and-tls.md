---
title: "04 — Ingress и TLS"
type: tutorial
tags: [kubernetes, tutorial, ingress, nginx, tls, cert-manager, letsencrypt, routing]
sources:
  docs: "https://kubernetes.io/docs/concepts/services-networking/ingress/"
related:
  - "[[kubernetes/tutorials/03-stateful-app]]"
  - "[[kubernetes/how-to/configure-networking]]"
  - "[[kubernetes/how-to/helm-basics]]"
---

# Tutorial 04 — Ingress и TLS

> **Цель:** Настроить Ingress для HTTP-маршрутизации по доменам и путям.
> Установить cert-manager для автоматических TLS-сертификатов (Let's Encrypt).

**Время:** ~40 минут
**Требования:** Пройден Tutorial 02. Minikube запущен.

## Шаг 1. Подготовка — два приложения

```yaml
# apps.yaml — два приложения для маршрутизации
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: "50m", memory: "32Mi" }
            limits: { cpu: "100m", memory: "64Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo
          args: ["-text=Hello from API"]
          ports:
            - containerPort: 5678
          resources:
            requests: { cpu: "50m", memory: "32Mi" }
            limits: { cpu: "100m", memory: "64Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 5678
```

```bash
kubectl apply -f apps.yaml
kubectl get pods,svc
```

## Шаг 2. Включить Ingress-контроллер

```bash
# Minikube
minikube addons enable ingress

# Проверить (подождать ~1 минуту)
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-xxx                1/1     Running

# IP Minikube
minikube ip
# 192.168.49.2
```

## Шаг 3. Ingress — маршрутизация по путям

```yaml
# ingress-path.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-path.yaml

# Проверить
kubectl get ingress
# NAME          CLASS   HOSTS   ADDRESS        PORTS
# app-ingress   nginx   *       192.168.49.2   80

# Тестировать
curl http://$(minikube ip)/
# → Nginx welcome page (frontend)

curl http://$(minikube ip)/api
# → Hello from API
```

## Шаг 4. Ingress — маршрутизация по доменам

```yaml
# ingress-host.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
    - host: api.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-host.yaml

# Добавить в /etc/hosts
echo "$(minikube ip) app.local api.local" | sudo tee -a /etc/hosts

# Тестировать
curl http://app.local
# → Nginx (frontend)

curl http://api.local
# → Hello from API
```

## Шаг 5. TLS с самоподписанным сертификатом

```bash
# Сгенерировать сертификат
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=app.local"

# Создать TLS Secret
kubectl create secret tls app-tls --cert=tls.crt --key=tls.key
```

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.local
      secretName: app-tls
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-tls.yaml

curl -k https://app.local
# → Nginx (через HTTPS, -k = игнорировать самоподписанный)
```

## Шаг 6. cert-manager (автоматические сертификаты)

Для production с Let's Encrypt:

```bash
# Установить cert-manager через Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set installCRDs=true
```

```yaml
# clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```yaml
# ingress с автоматическим TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls           # cert-manager создаст этот Secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

cert-manager автоматически получит сертификат от Let's Encrypt и обновит его перед истечением.

## Шаг 7. Очистка

```bash
kubectl delete ingress --all
kubectl delete -f apps.yaml
# Убрать записи из /etc/hosts
sudo sed -i '/app.local/d' /etc/hosts
```

## Что мы изучили

| Концепция | Что увидели |
|-----------|------------|
| Ingress Controller | nginx как единая точка входа в кластер |
| Path-based routing | `/` → frontend, `/api` → api |
| Host-based routing | `app.local` → frontend, `api.local` → api |
| TLS Secret | Самоподписанный сертификат через `kubectl create secret tls` |
| cert-manager | Автоматические Let's Encrypt сертификаты |
| Annotations | `rewrite-target`, `cluster-issuer` — настройка поведения Ingress |

## Цепочка tutorials завершена

```
01 Local Setup → 02 First Deployment → 03 Stateful App → 04 Ingress & TLS
```

Рекомендуемые следующие шаги:
- [[kubernetes/how-to/helm-basics]] — Helm для установки готовых решений
- [[kubernetes/how-to/troubleshoot]] — когда что-то пошло не так
- [[kubernetes/how-to/rbac]] — контроль доступа

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Ingress Controller не установлен | Ingress создан, но ADDRESS пустой | `minikube addons enable ingress` |
| `ingressClassName` не указан | Ingress не подхватывается контроллером | Добавить `ingressClassName: nginx` |
| /etc/hosts не обновлён | `curl: Could not resolve host` | Добавить `$(minikube ip) app.local` |
| cert-manager CRDs не установлены | `ClusterIssuer` не создаётся | `--set installCRDs=true` при helm install |
| Let's Encrypt rate limit | Ошибка получения сертификата | Использовать staging server для тестов |