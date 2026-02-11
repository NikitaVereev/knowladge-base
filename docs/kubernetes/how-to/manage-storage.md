---
title: "Управление хранилищем"
type: how-to
tags: [kubernetes, storage, pv, pvc, storageclass, configmap, secret, volumes]
sources:
  docs: "https://kubernetes.io/docs/concepts/storage/"
related:
  - "[[kubernetes/explanation/storage]]"
  - "[[kubernetes/how-to/manage-workloads]]"
  - "[[kubernetes/how-to/recipes/postgres-stateful]]"
---

# Управление хранилищем

> **TL;DR:** ConfigMap/Secret — конфигурация. PVC — запрос на диск (разработчик).
> StorageClass — динамическое создание дисков (админ). emptyDir — временное хранилище.

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_PORT: "3000"
  nginx.conf: |
    server {
      listen 80;
      location / { proxy_pass http://localhost:3000; }
    }
```

### Использование в Pod

```yaml
spec:
  containers:
    - name: app
      # Как переменные окружения
      envFrom:
        - configMapRef:
            name: app-config

      # Отдельные ключи
      env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_PORT

      # Как файл (volume)
      volumeMounts:
        - name: config-vol
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf              # только этот ключ
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

```bash
# Создание из файла
kubectl create configmap nginx-conf --from-file=nginx.conf

# Из литерала
kubectl create configmap app-env --from-literal=APP_ENV=prod --from-literal=PORT=3000
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                        # stringData = plain text (K8s закодирует)
  DB_USER: myapp
  DB_PASSWORD: SuperSecret123
```

```yaml
# В поде
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: DB_PASSWORD
```

```bash
# Создание
kubectl create secret generic db-creds \
  --from-literal=DB_USER=myapp \
  --from-literal=DB_PASSWORD=SuperSecret123

# TLS Secret
kubectl create secret tls app-tls \
  --cert=fullchain.pem --key=privkey.pem

# Docker registry Secret
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=user \
  --docker-password=token
```

> **Важно:** Secret хранятся в etcd в base64 (НЕ шифрование!). Для production включите Encryption at Rest или используйте external secrets (Vault, AWS Secrets Manager).

## PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce              # RWO = один узел
  storageClassName: standard     # имя StorageClass
  resources:
    requests:
      storage: 10Gi
```

### Монтирование в Pod

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: app-data
```

### Проверка

```bash
kubectl get pvc
# NAME       STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
# app-data   Bound    pvc-abc    10Gi       RWO            standard

kubectl get pv
# NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# pvc-abc   10Gi       RWO            Delete           Bound
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs    # или pd.csi.storage.gke.io
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Retain                  # Retain = данные сохраняются после удаления PVC
volumeBindingMode: WaitForFirstConsumer # создать диск когда под назначен на ноду
allowVolumeExpansion: true             # разрешить увеличение PVC
```

## Volumes (без PVC)

```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: shared
          mountPath: /shared
  initContainers:
    - name: init
      image: busybox
      command: ["sh", "-c", "echo ready > /shared/status"]
      volumeMounts:
        - name: shared
          mountPath: /shared
  volumes:
    # Временное хранилище (живёт пока жив Pod)
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi

    # Общее хранилище между контейнерами в поде
    - name: shared
      emptyDir: {}
```

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| Secret в plain YAML в Git | Секреты утекли | Использовать Sealed Secrets, SOPS, или external secret operator |
| PVC в `Pending` | `no persistent volumes available` | Проверить StorageClass существует. `kubectl describe pvc` |
| ConfigMap обновлён, под не видит | Env vars не обновляются без restart | `kubectl rollout restart deploy/web` после обновления CM |
| RWO-диск на multi-node | Pod `Pending` — volume уже на другой ноде | Для multi-node: RWX (NFS) или одна реплика |
| `reclaimPolicy: Delete` | Данные удалены после удаления PVC | `Retain` для production данных |
