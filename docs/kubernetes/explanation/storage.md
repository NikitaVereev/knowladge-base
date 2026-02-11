---
title: "Модель хранения данных"
type: explanation
tags: [kubernetes, storage, pv, pvc, storageclass, emptydir, configmap, secret, stateful]
sources:
  docs: "https://kubernetes.io/docs/concepts/storage/"
related:
  - "[[kubernetes/explanation/workload-resources]]"
  - "[[kubernetes/how-to/manage-storage]]"
  - "[[kubernetes/how-to/recipes/postgres-stateful]]"
---

# Модель хранения данных

> **TL;DR:** PVC (заявка) → StorageClass (провайдер) → PV (диск) — автоматически.
> emptyDir — временный, hostPath — опасный, PVC — production.
> RWO для обычных дисков, RWX для NFS/CephFS.

Контейнеры эфемерны — при перезапуске данные пропадают. Kubernetes предоставляет абстракцию, отделяющую **потребность** в хранилище от **физической реализации**.

## Типы Volumes

### emptyDir — временное хранилище

Создаётся при запуске пода, удаляется при его завершении. Общий для всех контейнеров в поде.

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 500Mi          # опционально
containers:
  - name: app
    volumeMounts:
      - name: cache
        mountPath: /tmp/cache
```

Когда использовать: кэш, temp-файлы, обмен данными между sidecar-контейнерами.

### hostPath — путь на ноде

Монтирует директорию **с хост-машины**. Опасен: привязывает под к конкретной ноде.

```yaml
volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```

Когда использовать: мониторинг (node_exporter), DaemonSet для логов. **Не для данных приложений.**

### ConfigMap и Secret

Конфиги и секреты как файлы или переменные окружения.

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secrets
containers:
  - name: app
    volumeMounts:
      - name: config
        mountPath: /etc/app/config
        readOnly: true
      - name: secrets
        mountPath: /etc/app/secrets
        readOnly: true
```

## PV / PVC / StorageClass

Три ключевые абстракции для persistent storage.

### PersistentVolume (PV) — «Сам диск»

Ресурс уровня кластера. Представляет реальное хранилище (AWS EBS, GCE PD, NFS, локальный SSD).

### PersistentVolumeClaim (PVC) — «Заявка на диск»

Ресурс в namespace. Запрос: «дай 10 GiB, SSD, ReadWriteOnce».

### StorageClass (SC) — «Тип услуги»

Описывает провайдера и параметры. При создании PVC с указанием StorageClass, диск создаётся **автоматически** (Dynamic Provisioning).

## Жизненный цикл

### Dynamic Provisioning (современный стандарт)

```
1. Admin создаёт StorageClass (один раз)
2. Dev создаёт PVC: «хочу 10Gi из класса standard»
3. K8s Provisioner:
   → вызывает API облака: «создай диск 10Gi»
   → создаёт PV
   → связывает PV ↔ PVC (Binding)
4. Pod монтирует PVC
```

```yaml
# StorageClass (создаёт admin)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com         # AWS EBS
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer

---
# PVC (создаёт разработчик)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 20Gi

---
# Pod (использует PVC)
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data       # ← ссылка на PVC
```

### Static Provisioning (старый подход, для NFS)

Admin вручную создаёт PV → Dev создаёт PVC → K8s находит подходящий PV и связывает.

## Access Modes

| Режим | Сокращение | Описание | Примеры |
|-------|-----------|----------|---------|
| ReadWriteOnce | **RWO** | R/W одной нодой | AWS EBS, GCE PD, Azure Disk |
| ReadWriteMany | **RWX** | R/W многими нодами | NFS, CephFS, EFS |
| ReadOnlyMany | **ROX** | Только чтение многими нодами | ConfigMap, shared data |
| ReadWriteOncePod | **RWOP** | R/W одним подом (K8s 1.27+) | Строгая изоляция |

> **Важно:** AWS EBS = только RWO. Для RWX нужен NFS-совместимый storage (EFS, CephFS).

## Reclaim Policy

Что происходит с PV после удаления PVC:

| Policy | Действие | Когда |
|--------|----------|-------|
| **Delete** | Диск удаляется вместе с данными | Default для облачных SC |
| **Retain** | PV остаётся в статусе `Released`, данные сохранены | Важные данные, требуется ручная очистка |

## volumeClaimTemplates (StatefulSet)

StatefulSet автоматически создаёт PVC для каждой реплики:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  serviceName: postgres
  template:
    spec:
      containers:
        - name: postgres
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:           # ← автоматическое создание PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast
        resources:
          requests:
            storage: 20Gi
```

Результат: `data-postgres-0`, `data-postgres-1`, `data-postgres-2` — отдельный PVC для каждого пода.

## Подводные камни

| Заблуждение | Реальность |
|------------|-----------|
| «hostPath — простой способ хранить данные» | Привязывает к ноде. Если под переехал — данные потеряны. Только для DaemonSet |
| «Можно смонтировать EBS на 2 ноды» | Нет. EBS — RWO. Для RWX — NFS (EFS) |
| «Delete reclaimPolicy — безопасно» | Удаление PVC → удаление диска → потеря данных. Для prod БД — `Retain` |
| «emptyDir сохраняется при перезапуске» | Только при рестарте контейнера внутри пода. При удалении/пересоздании пода — данные пропадают |
| «PVC можно уменьшить» | Только увеличить (`allowVolumeExpansion: true` в SC). Уменьшение не поддерживается |