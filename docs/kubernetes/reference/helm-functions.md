---
title: "Helm — Справочник функций"
type: reference
tags: [kubernetes, helm, functions, sprig, templates, cheatsheet]
sources:
  docs: "https://helm.sh/docs/chart_template_guide/function_list/"
  sprig: "https://masterminds.github.io/sprig/"
related:
  - "[[kubernetes/explanation/helm-functions]]"
  - "[[kubernetes/explanation/helm-objects]]"
  - "[[kubernetes/how-to/helm-basics]]"
---

# Helm — Справочник функций

> Ctrl+F — твой лучший друг здесь.

## Строковые

| Функция | Описание | Пример |
|---------|----------|--------|
| `upper` | В верхний регистр | `{{ "hello" \| upper }}` → `HELLO` |
| `lower` | В нижний регистр | `{{ "Hello" \| lower }}` → `hello` |
| `title` | Каждое слово с заглавной | `{{ "hello world" \| title }}` → `Hello World` |
| `trim` | Убрать пробелы по краям | `{{ " hi " \| trim }}` → `hi` |
| `trimSuffix` | Убрать суффикс | `{{ "app-" \| trimSuffix "-" }}` → `app` |
| `trimPrefix` | Убрать префикс | `{{ "v1.2.0" \| trimPrefix "v" }}` → `1.2.0` |
| `trunc` | Обрезать до N символов | `{{ "longname" \| trunc 4 }}` → `long` |
| `replace` | Замена подстроки | `{{ "foo_bar" \| replace "_" "-" }}` → `foo-bar` |
| `contains` | Проверка вхождения | `{{ if contains "tls" .Values.mode }}` |
| `hasPrefix` | Начинается с | `{{ if hasPrefix "https" .Values.url }}` |
| `hasSuffix` | Заканчивается на | `{{ if hasSuffix ".yaml" .name }}` |
| `repeat` | Повторить N раз | `{{ "ab" \| repeat 3 }}` → `ababab` |
| `substr` | Подстрока (start, end, str) | `{{ substr 0 3 "hello" }}` → `hel` |
| `nospace` | Убрать все пробелы | `{{ "h e l l o" \| nospace }}` → `hello` |
| `snakecase` | camelCase → snake_case | `{{ "userName" \| snakecase }}` → `user_name` |
| `camelcase` | snake_case → CamelCase | `{{ "user_name" \| camelcase }}` → `UserName` |
| `kebabcase` | camelCase → kebab-case | `{{ "userName" \| kebabcase }}` → `user-name` |
| `quote` | Обернуть в двойные кавычки | `{{ .Values.env \| quote }}` → `"production"` |
| `squote` | Обернуть в одинарные | `{{ .Values.env \| squote }}` → `'production'` |
| `cat` | Склеить через пробел | `{{ cat "hello" "world" }}` → `hello world` |
| `printf` | Форматирование | `{{ printf "%s:%s" .repo .tag }}` |
| `indent` | Отступ N пробелов | `{{ .data \| indent 4 }}` |
| `nindent` | Newline + отступ | `{{ .data \| nindent 4 }}` |

## Числовые

| Функция | Описание | Пример |
|---------|----------|--------|
| `int` | Привести к int | `{{ .Values.port \| int }}` |
| `int64` | Привести к int64 | `{{ .Values.size \| int64 }}` |
| `float64` | Привести к float | `{{ .Values.ratio \| float64 }}` |
| `add` | Сложение | `{{ add .Values.port 1 }}` |
| `sub` | Вычитание | `{{ sub 10 3 }}` → `7` |
| `mul` | Умножение | `{{ mul 3 4 }}` → `12` |
| `div` | Деление | `{{ div 10 3 }}` → `3` |
| `mod` | Остаток | `{{ mod 10 3 }}` → `1` |
| `max` | Максимум | `{{ max .Values.replicas 1 }}` |
| `min` | Минимум | `{{ min .Values.replicas 10 }}` |
| `ceil` | Округление вверх | `{{ ceil 1.1 }}` → `2` |
| `floor` | Округление вниз | `{{ floor 1.9 }}` → `1` |
| `round` | Округление (precision) | `{{ round 3.1416 2 }}` → `3.14` |

## Логические и сравнение

| Функция | Описание | Пример |
|---------|----------|--------|
| `default` | Значение по умолчанию | `{{ .Values.tag \| default "latest" }}` |
| `empty` | Проверка на пустоту | `{{ if empty .Values.host }}` |
| `not` | Отрицание | `{{ if not .Values.debug }}` |
| `and` | Логическое И | `{{ if and .Values.tls .Values.host }}` |
| `or` | Логическое ИЛИ | `{{ if or .Values.dev .Values.test }}` |
| `coalesce` | Первое непустое | `{{ coalesce .Values.tag .Chart.AppVersion "latest" }}` |
| `ternary` | Тернарный оператор | `{{ ternary "yes" "no" .Values.enabled }}` |
| `eq` | Равно | `{{ if eq .Values.env "prod" }}` |
| `ne` | Не равно | `{{ if ne .Values.env "test" }}` |
| `lt` / `le` | Меньше / меньше или равно | `{{ if lt .Values.replicas 3 }}` |
| `gt` / `ge` | Больше / больше или равно | `{{ if gt .Values.replicas 1 }}` |

## Списки

| Функция | Описание | Пример |
|---------|----------|--------|
| `list` | Создать список | `{{ list "a" "b" "c" }}` |
| `first` | Первый элемент | `{{ first .Values.hosts }}` |
| `last` | Последний элемент | `{{ last .Values.hosts }}` |
| `rest` | Все кроме первого | `{{ rest .Values.hosts }}` |
| `initial` | Все кроме последнего | `{{ initial .Values.hosts }}` |
| `append` | Добавить элемент | `{{ append .Values.list "new" }}` |
| `prepend` | Добавить в начало | `{{ prepend .Values.list "first" }}` |
| `concat` | Объединить списки | `{{ concat .Values.a .Values.b }}` |
| `has` | Элемент в списке | `{{ if has "admin" .Values.roles }}` |
| `without` | Список без элемента | `{{ without .Values.list "bad" }}` |
| `uniq` | Уникальные элементы | `{{ .Values.tags \| uniq }}` |
| `sortAlpha` | Сортировка строк | `{{ .Values.names \| sortAlpha }}` |
| `join` | Склеить в строку | `{{ .Values.args \| join "," }}` |
| `len` | Длина списка | `{{ len .Values.hosts }}` |
| `compact` | Убрать пустые элементы | `{{ .Values.list \| compact }}` |

## Словари

| Функция | Описание | Пример |
|---------|----------|--------|
| `dict` | Создать словарь | `{{ dict "key" "value" "k2" "v2" }}` |
| `get` | Получить по ключу | `{{ get .Values.config "db" }}` |
| `set` | Установить ключ | `{{ set .Values.labels "env" "prod" }}` |
| `unset` | Удалить ключ | `{{ unset .Values.labels "temp" }}` |
| `hasKey` | Проверить наличие ключа | `{{ if hasKey .Values.config "db" }}` |
| `keys` | Список ключей | `{{ keys .Values.config \| sortAlpha }}` |
| `values` | Список значений | `{{ values .Values.config }}` |
| `merge` | Слить (первый приоритетнее) | `{{ merge .Values.overrides .Values.defaults }}` |
| `mergeOverwrite` | Слить (последний приоритетнее) | `{{ mergeOverwrite .Values.defaults .Values.overrides }}` |
| `pick` | Выбрать ключи | `{{ pick .Values.all "a" "b" }}` |
| `omit` | Исключить ключи | `{{ omit .Values.all "secret" }}` |
| `deepCopy` | Глубокая копия | `{{ deepCopy .Values.config }}` |

## Преобразование типов

| Функция | Описание | Пример |
|---------|----------|--------|
| `toYaml` | Структура → YAML-строка | `{{ .Values.resources \| toYaml }}` |
| `toJson` | Структура → JSON | `{{ .Values.config \| toJson }}` |
| `toPrettyJson` | Структура → JSON с отступами | `{{ .Values.config \| toPrettyJson }}` |
| `toToml` | Структура → TOML | `{{ .Values.config \| toToml }}` |
| `fromYaml` | YAML-строка → структура | `{{ .data \| fromYaml }}` |
| `fromJson` | JSON-строка → структура | `{{ .data \| fromJson }}` |
| `toString` | Привести к строке | `{{ .Values.port \| toString }}` |
| `atoi` | Строка → int | `{{ "8080" \| atoi }}` |

## Криптография и кодирование

| Функция | Описание | Пример |
|---------|----------|--------|
| `b64enc` | Base64 encode | `{{ .Values.password \| b64enc }}` |
| `b64dec` | Base64 decode | `{{ .encoded \| b64dec }}` |
| `sha1sum` | SHA-1 хеш | `{{ .Values.data \| sha1sum }}` |
| `sha256sum` | SHA-256 хеш | `{{ .Values.data \| sha256sum }}` |
| `randAlphaNum` | Случайная строка (буквы + цифры) | `{{ randAlphaNum 16 }}` |
| `randAlpha` | Случайная строка (только буквы) | `{{ randAlpha 8 }}` |
| `randNumeric` | Случайная строка (только цифры) | `{{ randNumeric 6 }}` |
| `genPrivateKey` | RSA/ECDSA private key | `{{ genPrivateKey "rsa" }}` |
| `htpasswd` | Apache htpasswd | `{{ htpasswd "user" "pass" }}` |

## Валидация

| Функция | Описание | Пример |
|---------|----------|--------|
| `required` | Ошибка если пусто | `{{ required "image.tag required" .Values.image.tag }}` |
| `fail` | Явная ошибка рендеринга | `{{ fail "unsupported configuration" }}` |
| `kindOf` | Тип значения | `{{ kindOf .Values.port }}` → `float64` |
| `kindIs` | Проверка типа | `{{ if kindIs "string" .Values.tag }}` |
| `typeOf` | Go-тип значения | `{{ typeOf .Values.config }}` |

## Дата и время

| Функция | Описание | Пример |
|---------|----------|--------|
| `now` | Текущее время | `{{ now }}` |
| `date` | Форматирование | `{{ now \| date "2006-01-02" }}` |
| `dateModify` | Сдвиг даты | `{{ now \| dateModify "+2h" }}` |
| `toDate` | Парсинг строки | `{{ toDate "2006-01-02" "2026-03-15" }}` |
| `unixEpoch` | В Unix timestamp | `{{ now \| unixEpoch }}` |

> **Примечание:** Go использует reference-дату `Mon Jan 2 15:04:05 MST 2006` для форматирования. `2006-01-02` — не произвольный формат, а конкретные числа из reference-даты.

## Полезные однострочники

```yaml
# Имя ресурса (K8s лимит 63 символа, lowercase, без trailing -)
{{ printf "%s-%s" .Release.Name .Chart.Name | lower | trunc 63 | trimSuffix "-" }}

# Автоматический rollout при изменении ConfigMap
checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}

# Image с fallback на AppVersion
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"

# Все env из map
{{- range $k, $v := .Values.env }}
- name: {{ $k | upper }}
  value: {{ $v | quote }}
{{- end }}

# Опциональный блок целиком
{{- with .Values.tolerations }}
tolerations:
  {{- toYaml . | nindent 2 }}
{{- end }}
```
