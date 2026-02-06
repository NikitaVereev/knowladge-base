---
title: 2 Структура YAML манифестов
type: reference
tags: [kubernetes, yaml, spec, api]
---

# 2. Структура YAML манифестов

Любой объект в Kubernetes (Pod, Service, Deployment) описывается в формате YAML и состоит из четырех обязательных секций верхнего уровня.

---

## Базовая анатомия
```yaml
apiVersion: v1          # 1. Версия API
kind: Pod               # 2. Тип объекта
metadata:               # 3. Метаданные (имя, лейблы)
  name: my-app
  labels:
    app: backend
spec:                   # 4. Спецификация (Желаемое состояние)
  containers:
    - name: main
      image: nginx:1.19
```

### 1. apiVersion
Указывает версию схемы API для этого объекта.
*   `v1`: Core ресурсы (Pod, Service, ConfigMap, Secret, PVC).
*   `apps/v1`: Контроллеры (Deployment, DaemonSet, StatefulSet).
*   `networking.k8s.io/v1`: Сетевые ресурсы (Ingress).
*   `batch/v1`: Задачи (Job, CronJob).

### 2. kind
Тип создаваемого ресурса (регистр важен!): `Pod`, `Service`, `Deployment`, `Ingress`.

### 3. metadata
Данные для идентификации объекта.
*   `name`: Уникальное имя в пределах Namespace (обязательно).
*   `namespace`: Пространство имен (по умолчанию `default`).
*   `labels`: Ключ-значение теги для поиска и группировки (`app: web`, `env: prod`). Используются в селекторах.
*   `annotations`: Метаданные для систем/людей (автор, хеш коммита, настройки Ingress). Не используются для выборки.

### 4. spec
Самая важная часть. Описывает **желаемое состояние** (Desired State). Структура этой секции уникальна для каждого `kind`.

---

## Шпаргалка: Deployment Spec
Вложенная структура Deployment -> ReplicaSet -> Pod.

```yaml
spec:
  replicas: 3                   # Количество копий
  selector:                     # КАК найти свои поды
    matchLabels:
      app: my-app               # Должно совпадать с template.metadata.labels
  template:                     # Шаблон пода (Pod Template)
    metadata:
      labels:
        app: my-app             # Метка, которую получит под
    spec:                       # <-- Спецификация самого Пода начинается здесь
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
```

---

## Шпаргалка: Service Spec

```yaml
spec:
  type: ClusterIP               # ClusterIP (default), NodePort, LoadBalancer
  selector:                     # На какие поды слать трафик?
    app: my-app                 # Ищет поды с label "app: my-app"
  ports:
    - protocol: TCP
      port: 80                  # Порт самого Сервиса (Virtual IP)
      targetPort: 8080          # Порт внутри Контейнера
      nodePort: 30005           # (Только для NodePort) Порт на хосте
```

---

## Шпаргалка: Основные поля контейнера (внутри Pod spec)

```yaml
    spec:
      containers:
      - name: backend
        image: my-image:v1
        
        # Команды
        command: ["/bin/sh"]    # Аналог Docker ENTRYPOINT
        args: ["-c", "start.sh"]# Аналог Docker CMD
        
        # Ресурсы (Важно для Scheduler и OOMKiller)
        resources:
          requests:             # "Гарантированный минимум"
            memory: "64Mi"
            cpu: "250m"         # 0.25 ядра
          limits:               # "Жесткий потолок"
            memory: "128Mi"
            cpu: "500m"
            
        # Переменные окружения
        env:
        - name: DB_HOST
          value: "postgres"
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
              
        # Проверки здоровья (Probes)
        livenessProbe:          # "Жив ли процесс?" (Restart если нет)
          httpGet:
            path: /healthz
            port: 8080
        readinessProbe:         # "Готов принимать трафик?" (Remove from Service если нет)
          httpGet:
            path: /ready
            port: 8080
```
