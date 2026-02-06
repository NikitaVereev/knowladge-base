---
title: 1 Шпаргалка по командам kubectl
type: reference
tags: [kubernetes, kubectl, cli, cheatsheet]
---

# 1. Основные команды kubectl

`kubectl` — основной инструмент для управления кластером Kubernetes.
Общий синтаксис:
`kubectl [command] [TYPE] [NAME] [flags]`

---

## 1. Управление контекстами и конфигом
Команды для переключения между кластерами (например, Minikube vs Production).

| Команда | Описание |
| :--- | :--- |
| `kubectl config get-contexts` | Показать список доступных кластеров (контекстов). |
| `kubectl config current-context` | Показать текущий активный контекст. |
| `kubectl config use-context <name>` | Переключиться на другой кластер (например, `minikube`). |
| `kubectl cluster-info` | Показать адреса Master и CoreDNS. |

---

## 2. Основные операции с объектами (CRUD)
Базовые команды для работы с любыми ресурсами (Pod, Service, Deployment).

| Команда | Описание |
| :--- | :--- |
| `kubectl get <type>` | Список ресурсов в текущем namespace. |
| `kubectl get <type> -A` | Список ресурсов во **всех** namespace (All namespaces). |
| `kubectl get <type> -o wide` | Расширенный вывод (показывает IP, Node). |
| `kubectl get <type> <name> -o yaml` | Выгрузить полный манифест объекта в YAML. |
| `kubectl describe <type> <name>` | **Подробная информация** и события (Events) объекта. |
| `kubectl delete <type> <name>` | Удалить ресурс. |
| `kubectl delete -f file.yaml` | Удалить ресурсы, описанные в файле. |
| `kubectl apply -f file.yaml` | Создать или обновить ресурсы из файла (Декларативно). |

---

## 3. Отладка и Диагностика (Troubleshooting)
Самые важные команды, когда что-то сломалось.

| Команда | Описание |
| :--- | :--- |
| `kubectl logs <pod>` | Вывод логов контейнера (STDOUT). |
| `kubectl logs -f <pod>` | Чтение логов в реальном времени (stream). |
| `kubectl logs <pod> -c <container>` | Логи конкретного контейнера (если в поде их несколько). |
| `kubectl logs --previous <pod>` | Логи упавшего контейнера (полезно при CrashLoop). |
| `kubectl exec -it <pod> -- /bin/sh` | Зайти внутрь контейнера (интерактивная консоль). |
| `kubectl port-forward <pod> 8080:80` | Проброс порта: `localhost:8080` -> `pod:80`. |
| `kubectl get events --sort-by=.metadata.creationTimestamp` | Показать хронологию событий в кластере. |

---

## 4. Управление Приложениями (Deployments)

| Команда | Описание |
| :--- | :--- |
| `kubectl scale deploy/<name> --replicas=5` | Изменить количество реплик (масштабирование). |
| `kubectl rollout status deploy/<name>` | Следить за прогрессом обновления (выкатки). |
| `kubectl rollout history deploy/<name>` | Показать историю версий. |
| `kubectl rollout undo deploy/<name>` | Откатиться к предыдущей версии. |
| `kubectl rollout restart deploy/<name>` | Перезапустить поды (полезно при обновлении ConfigMap). |

---

## 5. Императивные команды (Быстрый старт)
Для быстрой генерации YAML или запуска тестов без создания файлов.

| Команда | Действие |
| :--- | :--- |
| `kubectl run nginx --image=nginx` | Запустить одиночный Pod. |
| `kubectl create deploy web --image=nginx` | Создать Deployment (управляет подами). |
| `kubectl expose deploy web --port=80 --type=NodePort` | Создать Service для Deployment. |
| **Генерация YAML (Dry Run):** | |
| `kubectl run nginx --image=nginx --dry-run=client -o yaml` | Показать YAML для Пода, не создавая его. |
| `kubectl create deploy web --image=nginx --replicas=3 --dry-run=client -o yaml` | Показать YAML для Deployment. |

---

## 6. Сокращения ресурсов (Aliases)
Kubernetes понимает короткие имена типов.

| Полное имя | Сокращение |
| :--- | :--- |
| `pods` | `po` |
| `services` | `svc` |
| `deployments` | `deploy` |
| `nodes` | `no` |
| `namespaces` | `ns` |
| `configmaps` | `cm` |
| `persistentvolumeclaims` | `pvc` |

*Пример:* `kubectl get po,svc` (получить и поды, и сервисы).
