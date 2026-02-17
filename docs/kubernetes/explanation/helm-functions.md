---
title: "Helm: функции и pipelines"
type: explanation
tags: [kubernetes, helm, templates, functions, pipelines, sprig, helpers, tpl]
sources:
  docs: "https://helm.sh/docs/chart_template_guide/function_list/"
  sprig: "https://masterminds.github.io/sprig/"
related:
  - "[[kubernetes/explanation/helm-objects]]"
  - "[[kubernetes/reference/helm-functions]]"
  - "[[kubernetes/how-to/helm-basics]]"
  - "[[kubernetes/reference/yaml-templates]]"
---

# Helm: функции и pipelines

> **TL;DR:** Helm-шаблоны используют Go template engine + библиотеку Sprig (~70 функций). Pipeline `{{ value | func1 | func2 }}` — цепочка трансформаций, где результат предыдущей функции передаётся последним аргументом следующей. `_helpers.tpl` — место для именованных шаблонов (`define`/`include`), переиспользуемых во всём чарте.

## Зачем это знать

Без функций шаблоны Helm — просто подстановка переменных. С функциями — полноценная генерация манифестов: валидация входных данных (`required`, `fail`), трансформация строк и структур (`toYaml`, `indent`), условная логика, переиспользуемые блоки. Разница между чартом, который ломается на каждом втором values.yaml, и чартом, который корректно обрабатывает edge cases.

## Pipelines

Pipeline — цепочка функций, соединённых `|`. Результат левой части передаётся **последним аргументом** правой:

```yaml
# Без pipeline (вложенные вызовы — читается тяжело)
{{ trunc 63 (trimSuffix "-" (printf "%s-%s" .Release.Name .Chart.Name)) }}

# С pipeline (читается как поток данных слева направо)
{{ printf "%s-%s" .Release.Name .Chart.Name | trimSuffix "-" | trunc 63 }}
```

Оба варианта эквивалентны. Pipeline предпочтителен — он читается линейно.

### Многошаговый pipeline

```yaml
# Сформировать имя ресурса: lowercase, обрезать до 63 символов (лимит K8s),
# убрать trailing дефис
{{ printf "%s-%s" .Release.Name .Chart.Name | lower | trunc 63 | trimSuffix "-" }}
```

Это настолько частый паттерн, что его выносят в `_helpers.tpl` (см. секцию «Именованные шаблоны»).

Helm использует Go template engine + библиотеку [Sprig](https://masterminds.github.io/sprig/) (~70 функций). Полный справочник всех функций с примерами — в [[kubernetes/reference/helm-functions]].

## toYaml + indent — самый частый паттерн

Проблема: `values.yaml` содержит вложенную структуру (resources, tolerations, nodeSelector), и её нужно вставить в шаблон как есть, с правильным отступом.

```yaml
# values.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 250m
    memory: 256Mi

tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

```yaml
# templates/deployment.yaml
spec:
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      tolerations:
        {{- toYaml .Values.tolerations | nindent 8 }}
```

Результат после рендеринга:

```yaml
spec:
  template:
    spec:
      containers:
        - name: payment-service
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "gpu"
          effect: "NoSchedule"
```

> **Важно:** `indent` добавляет отступ к уже существующему тексту. `nindent` добавляет перенос строки перед отступом. С `nindent` используй `{{-` (trim whitespace слева), иначе будет лишняя пустая строка.

### indent vs nindent

```yaml
# indent — НЕ добавляет newline, нужен перенос в шаблоне
data: |
{{ .Files.Get "config.yml" | indent 4 }}

# nindent — добавляет newline, используй с {{-
data:
  {{- .Files.Get "config.yml" | nindent 4 }}
```

## Управление пробелами

`{{-` и `-}}` — trim пробелов и переносов слева/справа от блока:

```yaml
# Без trim — лишние пустые строки в output
metadata:
  labels:
    {{ if .Values.team }}
    team: {{ .Values.team }}
    {{ end }}

# С trim — чистый output
metadata:
  labels:
    {{- if .Values.team }}
    team: {{ .Values.team }}
    {{- end }}
```

Правило: `{{-` ставь почти всегда на `if`, `range`, `end`, `define`, `include`. Не ставь на строках, которые должны рендерить значение с переносом.

## Именованные шаблоны (_helpers.tpl)

`define` создаёт переиспользуемый блок, `include` вызывает его. Все именованные шаблоны принято хранить в `_helpers.tpl` (файлы с `_` не рендерятся как манифесты).

### _helpers.tpl

```yaml
# templates/_helpers.tpl

# Имя ресурса: release-name + chart-name, обрезать до 63 символов
{{- define "myapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

# Стандартные labels (рекомендация Kubernetes docs)
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end -}}

# Selector labels (подмножество labels для matchLabels)
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

# Имя image с учётом registry
{{- define "myapp.image" -}}
{{- if .Values.image.registry -}}
  {{- printf "%s/%s:%s" .Values.image.registry .Values.image.repository .Values.image.tag -}}
{{- else -}}
  {{- printf "%s:%s" .Values.image.repository .Values.image.tag -}}
{{- end -}}
{{- end -}}
```

### Использование в шаблонах

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ include "myapp.image" . }}
```

### include vs template

```yaml
# template — вставляет как есть, НЕ работает с pipeline
{{ template "myapp.labels" . }}

# include — возвращает строку, РАБОТАЕТ с pipeline
{{ include "myapp.labels" . | nindent 4 }}
```

Всегда используй `include`. `template` не поддерживает pipeline, поэтому невозможно контролировать отступы — результат ломает YAML.

## Функция tpl — динамический рендеринг

`tpl` рендерит строку из `values.yaml` как Go template. Позволяет использовать шаблонные выражения внутри values.

```yaml
# values.yaml
config:
  greeting: "Hello from {{ .Release.Name }}"
  dbHost: "{{ .Release.Name }}-postgresql"
```

```yaml
# templates/configmap.yaml — БЕЗ tpl (строка как есть, не отрендерена)
data:
  greeting: {{ .Values.config.greeting }}
  # Результат: greeting: Hello from {{ .Release.Name }}  ← литерал, не значение

# С tpl (строка рендерится как шаблон)
data:
  greeting: {{ tpl .Values.config.greeting . }}
  dbHost: {{ tpl .Values.config.dbHost . }}
  # Результат: greeting: Hello from my-release
  #            dbHost: my-release-postgresql
```

> **Важно:** Второй аргумент `tpl` — контекст (обычно `.`). Без него шаблонные выражения внутри строки не имеют доступа к объектам.

## Условная логика и циклы

### if / else

```yaml
# Создать Ingress только если включён
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  rules:
    - host: {{ .Values.ingress.host | required "ingress.host is required when ingress is enabled" }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "myapp.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

### range — итерация

```yaml
# Несколько environment variables из map
env:
  {{- range $key, $value := .Values.env }}
  - name: {{ $key | upper }}
    value: {{ $value | quote }}
  {{- end }}
```

```yaml
# values.yaml
env:
  database_url: "postgres://db:5432/app"
  redis_url: "redis://cache:6379"
  log_level: "info"
```

Результат:

```yaml
env:
  - name: DATABASE_URL
    value: "postgres://db:5432/app"
  - name: LOG_LEVEL
    value: "info"
  - name: REDIS_URL
    value: "redis://cache:6379"
```

### with — сменить контекст

```yaml
# with перенаправляет . на указанный объект
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

Если `.Values.nodeSelector` пустой — весь блок не рендерится. Внутри `with` контекст `.` = `.Values.nodeSelector`, для доступа к корню используй `$`.

## Подводные камни

| Проблема | Симптом | Решение |
|----------|---------|---------|
| `toYaml` без `nindent` | Сломанный YAML — неправильные отступы | Всегда `toYaml .Values.x \| nindent N` |
| `template` вместо `include` | Невозможно контролировать отступы | Использовать `include` + `nindent` |
| `tpl` без второго аргумента | `nil pointer evaluating` | `tpl .Values.str .` — точка обязательна |
| Число из values рендерится как float | `port: 8.08e+03` | `{{ .Values.port \| int }}` или `"{{ .Values.port }}"` |
| `{{` без `-` в if/range | Пустые строки в output | `{{-` на управляющих конструкциях |
| `required` в optional-блоке | Ошибка даже когда блок не нужен | Обернуть в `if`: `{{- if .Values.x }}{{ required ... }}{{- end }}` |
| Имя define конфликтует с subchart | Subchart переопределяет шаблон | Префикс имени чарта: `define "myapp.labels"`, не `define "labels"` |

## Связанные материалы

- [[kubernetes/reference/helm-functions]] — справочник всех функций: строковые, числовые, списки, словари, крипто, валидация
- [[kubernetes/explanation/helm-objects]] — встроенные объекты: Release, Values, Chart, Files, Capabilities, Template
- [[kubernetes/how-to/helm-basics]] — установка Helm, CLI команды, values.yaml, создание чарта
- [[kubernetes/reference/yaml-templates]] — шаблоны K8s-манифестов
