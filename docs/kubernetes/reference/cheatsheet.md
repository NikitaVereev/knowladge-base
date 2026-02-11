---
title: "kubectl Cheatsheet"
type: reference
tags: [kubernetes, kubectl, cli, cheatsheet, aliases, context, debug, logs]
sources:
  docs: "https://kubernetes.io/docs/reference/kubectl/cheatsheet/"
related:
  - "[[kubernetes/reference/yaml-templates]]"
  - "[[kubernetes/reference/resource-reference]]"
  - "[[kubernetes/how-to/troubleshoot]]"
---

# Шпаргалка по kubectl

> **Справочник:** Все основные команды на одной странице. Ctrl+F для поиска.

## Контексты и конфиг

```bash
kubectl config get-contexts                    # список контекстов
kubectl config current-context                 # текущий контекст
kubectl config use-context minikube            # переключить контекст
kubectl config set-context --current --namespace=prod  # namespace по умолчанию
kubectl cluster-info                           # адреса API Server, CoreDNS
```

## CRUD операции

```bash
# Список ресурсов
kubectl get pods                               # поды в текущем ns
kubectl get pods -A                            # во всех namespace
kubectl get pods -o wide                       # + IP, Node
kubectl get pods -o yaml                       # полный YAML
kubectl get pods -l app=web                    # по label
kubectl get pods --field-selector status.phase=Running
kubectl get po,svc,deploy                      # несколько типов

# Детальная информация
kubectl describe pod web-abc123                # события + состояние
kubectl describe node node-1                   # ресурсы, taints, conditions

# Создание / обновление
kubectl apply -f manifest.yaml                 # декларативно (create/update)
kubectl apply -f k8s/ --recursive              # из директории
kubectl create -f manifest.yaml                # императивно (только create)

# Удаление
kubectl delete pod web-abc123
kubectl delete -f manifest.yaml                # удалить по файлу
kubectl delete pods --all -n staging           # все поды в namespace
kubectl delete pods -l app=test                # по label
```

## Отладка и диагностика

```bash
# Логи
kubectl logs web-abc123                        # логи пода
kubectl logs -f web-abc123                     # follow (stream)
kubectl logs web-abc123 -c sidecar             # конкретный контейнер
kubectl logs --previous web-abc123             # логи предыдущего инстанса (CrashLoop)
kubectl logs -l app=web --all-containers       # логи по label

# Shell / exec
kubectl exec -it web-abc123 -- /bin/sh         # зайти в контейнер
kubectl exec web-abc123 -- cat /etc/config     # выполнить команду

# Port forwarding
kubectl port-forward pod/web-abc123 8080:80    # pod
kubectl port-forward svc/web 8080:80           # service
kubectl port-forward deploy/web 8080:80        # deployment

# Копирование файлов
kubectl cp web-abc123:/var/log/app.log ./app.log
kubectl cp ./config.yml web-abc123:/etc/app/

# События
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -w                          # watch в реальном времени

# Статус ноды
kubectl top nodes                              # CPU/RAM нод (metrics-server)
kubectl top pods                               # CPU/RAM подов
kubectl top pods --sort-by=memory

# Debug pod (запустить отладочный контейнер)
kubectl debug -it web-abc123 --image=busybox --target=app
kubectl run debug --rm -it --image=busybox -- sh   # временный под
```

## Deployments

```bash
kubectl scale deploy/web --replicas=5          # масштабирование
kubectl rollout status deploy/web              # прогресс обновления
kubectl rollout history deploy/web             # история ревизий
kubectl rollout undo deploy/web                # откат к предыдущей
kubectl rollout undo deploy/web --to-revision=3  # к конкретной
kubectl rollout restart deploy/web             # пересоздать поды
kubectl set image deploy/web app=myapp:v2      # обновить image
```

## Namespace

```bash
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging               # УДАЛИТ ВСЕ ресурсы внутри!
kubectl get all -n kube-system                 # системные компоненты
```

## Генерация YAML (Dry Run)

```bash
# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml

# Deployment
kubectl create deploy web --image=nginx --replicas=3 \
  --dry-run=client -o yaml

# Service
kubectl expose deploy web --port=80 --type=ClusterIP \
  --dry-run=client -o yaml

# Job
kubectl create job migrate --image=myapp -- python manage.py migrate \
  --dry-run=client -o yaml

# ConfigMap
kubectl create configmap app-config --from-file=config.yml \
  --dry-run=client -o yaml

# Secret
kubectl create secret generic db-creds \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml

# Подсмотреть YAML существующего ресурса
kubectl get deploy web -o yaml
```

## ConfigMap и Secret

```bash
# Создать ConfigMap
kubectl create configmap app-config \
  --from-file=config.yml \
  --from-literal=DB_HOST=postgres

# Создать Secret
kubectl create secret generic db-creds \
  --from-literal=password=secret123

# Просмотреть (Secret показывает base64)
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

## Labels и Selectors

```bash
# Добавить label
kubectl label pod web-abc123 env=prod

# Удалить label
kubectl label pod web-abc123 env-

# Фильтрация
kubectl get pods -l app=web
kubectl get pods -l "app=web,env=prod"
kubectl get pods -l "app in (web, api)"
kubectl get pods -l "env!=test"
```

## Сокращения ресурсов (Aliases)

| Ресурс | Alias | | Ресурс | Alias |
|--------|-------|-|--------|-------|
| pods | **po** | | configmaps | **cm** |
| services | **svc** | | persistentvolumeclaims | **pvc** |
| deployments | **deploy** | | persistentvolumes | **pv** |
| replicasets | **rs** | | storageclasses | **sc** |
| statefulsets | **sts** | | serviceaccounts | **sa** |
| daemonsets | **ds** | | namespaces | **ns** |
| jobs | **job** | | nodes | **no** |
| cronjobs | **cj** | | events | **ev** |
| ingresses | **ing** | | endpoints | **ep** |
| networkpolicies | **netpol** | | horizontalpodautoscalers | **hpa** |

```bash
# Полный список для вашего кластера
kubectl api-resources
```

## Полезные однострочники

```bash
# Все поды не в Running
kubectl get pods -A --field-selector status.phase!=Running

# Restart всех подов deployment
kubectl rollout restart deploy -n default

# Удалить Evicted поды
kubectl get pods -A | grep Evicted | awk '{print $2 " -n " $1}' | xargs -L1 kubectl delete pod

# Получить images всех подов
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Ресурсы ноды (requests/limits)
kubectl describe nodes | grep -A5 "Allocated resources"

# Создать pod для DNS-отладки
kubectl run dnsutils --image=gcr.io/kubernetes-e2e-test-images/dnsutils --rm -it -- nslookup kubernetes
```