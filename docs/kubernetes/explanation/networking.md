---
title: "Сетевая модель"
type: explanation
tags: [kubernetes, networking, service, clusterip, nodeport, loadbalancer, ingress, dns, cni]
sources:
  docs: "https://kubernetes.io/docs/concepts/services-networking/"
related:
  - "[[kubernetes/explanation/architecture]]"
  - "[[kubernetes/how-to/configure-networking]]"
  - "[[kubernetes/tutorials/04-ingress-and-tls]]"
---

# Сетевая модель Kubernetes

> **TL;DR:** Каждый Pod получает уникальный IP. Service — стабильный фасад для группы подов.
> ClusterIP — внутренний, NodePort — через ноду, LoadBalancer — через облако, Ingress — HTTP-роутинг по доменам.

Поды эфемерны — при перезапуске меняют IP. Service — абстракция, дающая стабильный endpoint и балансировку.

## Фундаментальные правила

1. Каждый Pod получает **собственный IP-адрес** (не нужен NAT между подами)
2. Поды на разных нодах могут общаться напрямую по IP
3. Контейнеры внутри одного пода делят `localhost`
4. За сетевую связность отвечает **CNI-плагин** (Calico, Cilium, Flannel)

## Типы Service

### ClusterIP (по умолчанию)

Виртуальный IP, доступный **только изнутри кластера**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP              # можно не указывать — это default
  selector:
    app: backend               # направлять трафик на поды с label app=backend
  ports:
    - port: 80                 # порт Service
      targetPort: 8080         # порт контейнера
```

```
[Frontend Pod] → backend:80 → [kube-proxy/iptables] → [Backend Pod :8080]
```

Когда использовать: микросервисы друг с другом, БД внутри кластера.

### NodePort

Открывает порт (30000-32767) на **каждой ноде** кластера.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080          # опционально, иначе рандомный
```

```
User → <Node-IP>:30080 → Service → Pod :8080
```

Когда использовать: dev/test, on-premise без cloud LB, Minikube.

### LoadBalancer

Создаёт внешний балансировщик через API облачного провайдера.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

```
User → [Cloud LB (public IP)] → NodePort → Service → Pod
```

Когда использовать: production в облаке (AWS ALB/NLB, GCP LB, Azure LB).

### Headless Service (`ClusterIP: None`)

Без виртуального IP — DNS возвращает IP-адреса **всех подов** напрямую.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None              # ← headless
  selector:
    app: postgres
  ports:
    - port: 5432
```

Когда использовать: StatefulSet (БД, Kafka, Elasticsearch), когда подам нужно знать друг друга по имени.

DNS для Headless + StatefulSet: `postgres-0.postgres.default.svc.cluster.local`

## Сводная таблица

| Тип | Доступность | Для чего |
|-----|------------|----------|
| **ClusterIP** | Внутри кластера | Backend, DB, внутренние API |
| **NodePort** | Извне по IP ноды + порт 3xxxx | Dev/test, on-premise |
| **LoadBalancer** | Публичный IP (облако) | Production в облаке |
| **Headless** | Внутри кластера, прямые IP подов | StatefulSet, БД |

## Ingress

Ingress — **не тип Service**. Это HTTP(S)-маршрутизатор, раскидывающий трафик по ClusterIP-сервисам на основе доменов и path.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

```
User → [Ingress Controller (Nginx/Traefik)] → ClusterIP Service → Pod
                  ↑
          Обычно за LoadBalancer Service
```

Требуется **Ingress Controller** — отдельный под, который читает Ingress-ресурсы и конфигурирует прокси. Популярные: `ingress-nginx`, `traefik`, `kong`.

## DNS внутри кластера

CoreDNS автоматически создаёт DNS-записи для Service:

| Формат | Пример |
|--------|--------|
| `<service>.<namespace>.svc.cluster.local` | `backend.default.svc.cluster.local` |
| `<service>.<namespace>` | `backend.default` |
| `<service>` (в том же namespace) | `backend` |

Из пода в том же namespace достаточно: `curl http://backend:80`

## NetworkPolicy

Ограничение сетевого трафика между подами (firewall уровня кластера):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
```

По умолчанию все поды могут общаться с кем угодно. NetworkPolicy — opt-in (добавляет ограничения).

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «Ingress = LoadBalancer» | Ingress — L7 (HTTP), LoadBalancer — L4 (TCP). Ingress обычно работает *поверх* LoadBalancer |
| «NodePort годится для prod» | Неудобно: порты 30000+, нужно знать IP нод. Для prod — LoadBalancer или Ingress |
| «Pod IP стабилен» | Нет. При перезапуске Pod получает новый IP. Только Service даёт стабильный endpoint |
| «NetworkPolicy работает из коробки» | Нужен CNI-плагин с поддержкой (Calico, Cilium). Flannel не поддерживает |