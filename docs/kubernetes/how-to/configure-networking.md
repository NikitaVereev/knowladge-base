---
title: "Настройка сети"
type: how-to
tags: [kubernetes, networking, service, ingress, nginx, networkpolicy, dns, port-forward]
sources:
  docs: "https://kubernetes.io/docs/concepts/services-networking/"
related:
  - "[[kubernetes/explanation/networking]]"
  - "[[kubernetes/how-to/manage-workloads]]"
  - "[[kubernetes/tutorials/04-ingress-and-tls]]"
---

# Настройка сети

> **TL;DR:** Service = стабильный endpoint для подов. Ingress = HTTP-маршрутизация по доменам.
> NetworkPolicy = firewall между подами. DNS: `<svc>.<ns>.svc.cluster.local`.

## Service

### ClusterIP (по умолчанию — только внутри кластера)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80               # порт Service
      targetPort: 8080        # порт контейнера
```

### NodePort (доступ снаружи через порт ноды)

```yaml
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080         # 30000-32767
```

### LoadBalancer (облако — внешний IP)

```yaml
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
# В облаке: получит external IP. Локально: minikube tunnel
```

### Headless Service (для StatefulSet)

```yaml
spec:
  clusterIP: None             # без виртуального IP
  selector:
    app: postgres
  ports:
    - port: 5432
# DNS вернёт IP всех подов, а не один виртуальный
```

## Ingress

HTTP/HTTPS-маршрутизация. Требует Ingress-контроллера (nginx, traefik).

```bash
# Minikube: включить Ingress
minikube addons enable ingress
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
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
                name: backend
                port:
                  number: 8080
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-panel
                port:
                  number: 80
```

### Ingress с TLS

```yaml
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-secret     # TLS Secret
  rules:
    - host: app.example.com
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

Создать TLS Secret:

```bash
kubectl create secret tls app-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem
```

## DNS внутри кластера

```
<service>.<namespace>.svc.cluster.local

# Примеры:
backend.default.svc.cluster.local       # полный FQDN
backend.default                          # сокращённый (внутри кластера)
backend                                  # если в том же namespace
```

Для StatefulSet с Headless Service:

```
<pod-name>.<service>.<namespace>.svc.cluster.local
postgres-0.postgres.default.svc.cluster.local
```

## NetworkPolicy (firewall)

По умолчанию все поды могут общаться со всеми. NetworkPolicy ограничивает доступ.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend                # применяется к подам backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend       # разрешить входящий трафик только от frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres       # разрешить исходящий только к postgres
      ports:
        - port: 5432
```

## Быстрый доступ для отладки

```bash
# Port-forward к поду
kubectl port-forward pod/web-abc12 8080:80

# Port-forward к Service
kubectl port-forward svc/backend 8080:80

# Minikube: открыть Service в браузере
minikube service web-service

# Minikube: tunnel для LoadBalancer
minikube tunnel
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Service selector не совпадает с pod labels | `kubectl get endpoints svc` — пусто | Проверить labels: `kubectl get pods --show-labels` |
| Ingress без Ingress-контроллера | Ingress создан, но не работает | `minikube addons enable ingress` или установить nginx-ingress |
| `port` vs `targetPort` путаница | Подключение отклонено | `port` = порт Service, `targetPort` = порт контейнера |
| NetworkPolicy без CNI поддержки | Policy создана, но не применяется | Minikube: `--cni=calico`. Проверить CNI-плагин кластера |
| Headless Service с selector = пустой | DNS не резолвит поды | Указать `selector` и убедиться что поды есть |
