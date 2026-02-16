---
title: "Helm: встроенные объекты шаблонов"
type: explanation
tags: [kubernetes, helm, templates, release, values, chart, capabilities, go-templates]
sources:
  docs: "https://helm.sh/docs/chart_template_guide/builtin_objects/"
related:
  - "[[kubernetes/how-to/helm-basics]]"
  - "[[kubernetes/reference/yaml-templates]]"
  - "[[kubernetes/how-to/recipes/web-app-deploy]]"
---

# Helm: встроенные объекты шаблонов

> **TL;DR:** В шаблонах Helm доступны 6 корневых объектов: `Release` (мета релиза), `Values` (пользовательские параметры), `Chart` (метаданные чарта), `Files` (доступ к файлам), `Capabilities` (возможности кластера), `Template` (текущий шаблон). Все данные в шаблоне доступны через `{{ .ObjectName.Field }}`.

## Зачем это знать

Шаблоны Helm — это Go templates с доступом к контексту. Каждый `.yaml` в `templates/` получает корневой объект `.` (dot), через который доступны все встроенные объекты. Без понимания этих объектов шаблоны превращаются в копипасту примеров из Stack Overflow — работает, но непонятно почему.

## Корневой контекст

Внутри любого шаблона `.` (dot) — это корневой объект, содержащий все встроенные объекты:

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # .Release, .Chart, .Values — всё доступно через dot
  name: {{ .Release.Name }}-config
  labels:
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
data:
  environment: {{ .Values.env }}
```

> **Важно:** Внутри `range` и `with` контекст `.` меняется. Чтобы обратиться к корню — используй `$`:
> ```yaml
> {{- range .Values.servers }}
>   # здесь . = текущий элемент, а не корневой объект
>   name: {{ $.Release.Name }}-{{ .name }}
> {{- end }}
> ```

## Release — информация о релизе

Данные о текущем процессе установки/обновления. Заполняется Helm автоматически, не зависит от `values.yaml`.

| Поле | Тип | Описание |
|------|-----|----------|
| `.Release.Name` | string | Имя релиза (`helm install **my-app** ...`) |
| `.Release.Namespace` | string | Namespace, в который устанавливается |
| `.Release.Revision` | int | Номер ревизии (1 при install, инкрементируется при upgrade) |
| `.Release.IsUpgrade` | bool | `true` если это `helm upgrade` |
| `.Release.IsInstall` | bool | `true` если это `helm install` |
| `.Release.Service` | string | Всегда `"Helm"` |

```yaml
# templates/deployment.yaml
metadata:
  name: {{ .Release.Name }}-api
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    # Полезно для отслеживания: какая ревизия задеплоена
    helm.sh/revision: "{{ .Release.Revision }}"
```

**Типичное использование:** формирование уникальных имён ресурсов (`{{ .Release.Name }}-component`), чтобы несколько релизов одного чарта не конфликтовали в кластере.

## Values — пользовательские параметры

Объединённые значения из нескольких источников (в порядке приоритета):

1. `values.yaml` в чарте (defaults)
2. Файл через `-f custom-values.yaml`
3. Параметры через `--set key=value` (наивысший приоритет)

```yaml
# values.yaml
app:
  name: payment-service
  replicas: 3
  image:
    repository: myregistry/payment
    tag: "1.4.2"
  config:
    logLevel: info
    dbPool: 10
```

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.app.replicas }}
  template:
    spec:
      containers:
        - name: {{ .Values.app.name }}
          image: {{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.app.config.logLevel }}
            - name: DB_POOL_SIZE
              value: "{{ .Values.app.config.dbPool }}"
```

### Защита от отсутствующих значений

```yaml
# default — значение по умолчанию, если ключ не задан
replicas: {{ .Values.app.replicas | default 1 }}

# required — шаблон не отрендерится без этого значения
image: {{ required "app.image.repository is required" .Values.app.image.repository }}

# Условный блок
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}
```

## Chart — метаданные из Chart.yaml

Содержимое файла `Chart.yaml`. Доступны все поля спецификации.

| Поле | Описание |
|------|----------|
| `.Chart.Name` | Имя чарта |
| `.Chart.Version` | Версия чарта (SemVer) |
| `.Chart.AppVersion` | Версия приложения внутри чарта |
| `.Chart.Description` | Описание |
| `.Chart.Type` | `application` или `library` |
| `.Chart.Home` | URL домашней страницы проекта |

```yaml
# Chart.yaml
apiVersion: v2
name: payment-service
version: 0.3.1
appVersion: "1.4.2"
description: Payment processing microservice
```

```yaml
# templates/deployment.yaml
metadata:
  labels:
    # Стандартные метки Kubernetes по рекомендации docs
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
```

> **Примечание:** `.Chart.Version` — версия чарта (пакета), `.Chart.AppVersion` — версия приложения. Они могут не совпадать: обновление values.yaml без изменения приложения бампит только `Chart.Version`.

## Files — доступ к файлам чарта

Доступ к любым файлам внутри чарта, **кроме**: `templates/`, `Chart.yaml`, `values.yaml`, `.helmignore`.

| Метод | Описание |
|-------|----------|
| `.Files.Get "path"` | Содержимое файла как строка |
| `.Files.GetBytes "path"` | Содержимое как `[]byte` |
| `.Files.Glob "pattern"` | Файлы по паттерну |
| `.Files.AsConfig` | Содержимое как блок `data:` для ConfigMap |
| `.Files.AsSecrets` | Содержимое как base64 для Secret |
| `.Files.Lines "path"` | Файл построчно |

```
my-chart/
├── Chart.yaml
├── values.yaml
├── config/
│   ├── app.conf        # ← доступен через .Files
│   └── nginx.conf      # ← доступен через .Files
├── scripts/
│   └── init.sql        # ← доступен через .Files
└── templates/
    └── configmap.yaml
```

```yaml
# templates/configmap.yaml — загрузить конфиги из файлов
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  # Один файл
  app.conf: |
{{ .Files.Get "config/app.conf" | indent 4 }}

  # Все файлы из директории
{{ (.Files.Glob "config/*").AsConfig | indent 2 }}
```

```yaml
# templates/secret.yaml — файлы как Secret (auto base64)
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-certs
type: Opaque
data:
{{ (.Files.Glob "certs/*").AsSecrets | indent 2 }}
```

## Capabilities — информация о кластере

Информация о возможностях Kubernetes-кластера, в который происходит установка. Позволяет адаптировать шаблоны под версию кластера.

| Поле | Описание |
|------|----------|
| `.Capabilities.KubeVersion` | Версия Kubernetes (`v1.29.0`) |
| `.Capabilities.KubeVersion.Major` | Мажорная версия (`1`) |
| `.Capabilities.KubeVersion.Minor` | Минорная версия (`29`) |
| `.Capabilities.APIVersions` | Список поддерживаемых API |
| `.Capabilities.APIVersions.Has "api/v1"` | Проверка конкретного API |
| `.Capabilities.HelmVersion` | Версия Helm |

```yaml
# Выбор apiVersion в зависимости от версии кластера
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
kind: Ingress
```

```yaml
# Использование CRD только если API доступен
{{- if .Capabilities.APIVersions.Has "monitoring.coreos.com/v1" }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}-monitor
spec:
  selector:
    matchLabels:
      app: {{ .Release.Name }}
{{- end }}
```

**Типичное использование:** обратная совместимость чартов с разными версиями кластеров, условное создание ресурсов (ServiceMonitor только если Prometheus Operator установлен).

## Template — информация о текущем шаблоне

Мета-информация о файле шаблона, который сейчас рендерится.

| Поле | Описание |
|------|----------|
| `.Template.Name` | Путь к файлу (`mychart/templates/deployment.yaml`) |
| `.Template.BasePath` | Директория (`mychart/templates`) |

```yaml
# Добавить аннотацию, показывающую из какого шаблона сгенерирован ресурс
metadata:
  annotations:
    helm.sh/template: {{ .Template.Name }}
```

Используется редко — в основном для дебага сложных чартов с большим количеством шаблонов, чтобы понять какой файл сгенерировал конкретный ресурс.

## Как объекты взаимодействуют

Пример шаблона, использующего все 6 объектов:

```yaml
# templates/deployment.yaml
{{- if .Capabilities.APIVersions.Has "apps/v1" }}
apiVersion: apps/v1
{{- end }}
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
  annotations:
    helm.sh/template: {{ .Template.Name }}
spec:
  replicas: {{ .Values.replicas | default 1 }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          volumeMounts:
            - name: config
              mountPath: /etc/app
      volumes:
        - name: config
          configMap:
            name: {{ .Release.Name }}-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
{{ (.Files.Glob "config/*").AsConfig | indent 2 }}
```

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| `.Values.port` рендерится как float | `port: 8080` → `port: 8.08e+03` | Приведение: `{{ int .Values.port }}` или кавычки: `"{{ .Values.port }}"` |
| `.` теряется внутри `range`/`with` | `{{ .Release.Name }}` — пусто | Использовать `$`: `{{ $.Release.Name }}` |
| Файл не виден через `.Files` | Пустой вывод | Файл в `templates/` или в `.helmignore` → недоступен |
| `nil pointer` при отсутствующем ключе | Ошибка рендеринга | `{{ .Values.key | default "fallback" }}` или `{{- if .Values.key }}` |
| `.Chart.AppVersion` без кавычек | YAML парсит `1.0` как число | Всегда `"{{ .Chart.AppVersion }}"` в кавычках |
| `.Capabilities` врёт при `helm template` | Локальный рендер без кластера | `helm template` использует дефолтные Capabilities. Проверять через `helm install --dry-run` |

## Связанные материалы

- [[kubernetes/how-to/helm-basics]] — установка, CLI команды, values.yaml
- [[kubernetes/reference/yaml-templates]] — шаблоны K8s-манифестов
- [[kubernetes/how-to/recipes/web-app-deploy]] — полный набор манифестов
