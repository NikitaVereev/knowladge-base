---
title: 3 Сокращения ресурсов (Aliases)
type: reference
tags: [kubernetes, kubectl, aliases, shortnames, api]
---

# 3. Сокращения ресурсов (Shortnames)

Kubernetes API поддерживает короткие псевдонимы (aliases) для большинства типов ресурсов. Их использование значительно ускоряет работу в консоли `kubectl`.

Вам не нужно писать `kubectl get pods`, достаточно `kubectl get po`.

---

## Частые ресурсы (Workloads & Networking)

| Полное имя (Kind) | Сокращение | Русское название |
| :--- | :--- | :--- |
| `pods` | **`po`** | Под |
| `services` | **`svc`** | Сервис |
| `deployments` | **`deploy`** | Деплоймент |
| `replicasets` | **`rs`** | РепликаСет |
| `statefulsets` | **`sts`** | СтейтфулСет |
| `daemonsets` | **`ds`** | ДемонСет |
| `jobs` | **`job`** | Задача |
| `cronjobs` | **`cj`** | Периодическая задача |
| `ingresses` | **`ing`** | Ингресс (Вход) |

---

## Конфигурация и Хранение (Config & Storage)

| Полное имя (Kind) | Сокращение | Русское название |
| :--- | :--- | :--- |
| `configmaps` | **`cm`** | КонфигМап |
| `secrets` | *нет* | Секрет (пишется полностью) |
| `persistentvolumeclaims` | **`pvc`** | Заявка на том |
| `persistentvolumes` | **`pv`** | Постоянный том |
| `storageclasses` | **`sc`** | Класс хранилища |
| `serviceaccounts` | **`sa`** | Сервисный аккаунт |

---

## Кластерные ресурсы (Cluster & Nodes)

| Полное имя (Kind) | Сокращение | Русское название |
| :--- | :--- | :--- |
| `nodes` | **`no`** | Нода (Узел) |
| `namespaces` | **`ns`** | Пространство имен |
| `events` | **`ev`** | События |
| `endpoints` | **`ep`** | Эндпоинты (IP адреса подов) |
| `componentstatuses` | **`cs`** | Статус компонентов |
| `networkpolicies` | **`netpol`** | Сетевая политика |

---

## Как узнать сокращение?
Если вы забыли сокращение или работаете с кастомным ресурсом (CRD), вы можете запросить полный список поддерживаемых сокращений у вашего кластера:

```bash
kubectl api-resources
```

Вывод покажет колонку `SHORTNAMES`:
```text
NAME          SHORTNAMES   APIVERSION    NAMESPACED   KIND
pods          po           v1            true         Pod
services      svc          v1            true         Service
...
```
