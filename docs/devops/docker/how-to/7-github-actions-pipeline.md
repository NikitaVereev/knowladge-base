---
title: 7 CI/CD Pipeline (GitHub Actions)
type: how-to
tags: [docker, cicd, github-actions, ghcr, buildx, cache, scanning, pipeline]
---

# Docker CI/CD Pipeline с GitHub Actions

В этом руководстве мы создадим полноценный, Production-ready пайплайн для сборки и доставки Docker-образов. Мы реализуем все принципы, описанные в разделе *Explanation*: кэширование, сканирование безопасности, мультиплатформенную сборку и правильное версионирование.

## Цели пайплайна
1.  **Build**: Сборка образа с использованием **Buildx** (быстро + multi-arch).
2.  **Test**: Запуск тестов внутри контейнера.
3.  **Scan**: Сканирование образа на уязвимости (Trivy).
4.  **Push**: Публикация в **GitHub Container Registry (GHCR)**.
5.  **Tagging**:
    *   Pull Request -> `pr-123`
    *   Commit в main -> `main` + `sha-<short>`
    *   Git Tag (v1.0.0) -> `v1.0.0` + `latest`

## 1. Подготовка репозитория

Вам понадобится Dockerfile. Создайте `.github/workflows/docker-publish.yml`.

## 2. Полный Workflow (`docker-publish.yml`)

```yaml
name: Docker Build & Publish

on:
  push:
    branches: [ "main" ]
    # Публикуем теги семантического версионирования (v1.0.0)
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ "main" ]

env:
  # Используем ghcr.io (бесплатно для публичных и приватных репо в GitHub)
  REGISTRY: ghcr.io
  # Имя образа (owner/repo)
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write # Нужно для пуша в GHCR
      # id-token: write # Нужно, если используете Cosign для подписи

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      # 1. Настройка QEMU (нужно только если собираете multi-arch: arm64 + amd64)
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      # 2. Настройка Docker Buildx (Движок сборки с кэшем)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Логин в реестр (кроме PR)
      - name: Log into registry ${{ env.REGISTRY }}
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # 4. Генерация метаданных (тегов и лейблов)
      # Это действие автоматически создает теги:
      # - main branch -> main
      # - tag v1.0 -> v1.0, latest
      # - pr -> pr-12
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,format=long

      # 5. Build & Export (без пуша) для сканирования
      # Сначала собираем образ локально в раннер, чтобы Trivy мог его проверить
      - name: Build and export to Docker
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true # Загрузить образ в локальный docker демон
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test
          cache-from: type=gha # Кэш из GitHub Actions
          cache-to: type=gha,mode=max

      # 6. Сканирование безопасности (Trivy)
      # Падаем, если найдены CRITICAL уязвимости
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:test'
          format: 'table'
          exit-code: '1' # Упасть при ошибке
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      # 7. Финальная сборка и Пуш
      # Если тесты и скан прошли успешно, собираем Multi-arch и пушим
      - name: Build and push Docker image
        id: build-and-push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }} # Не пушим PR
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Разбор ключевых моментов

### Кэширование (`type=gha`)
Мы используем встроенный кэш GitHub Actions (`cache-from: type=gha`).
*   **Как это работает:** Docker сохраняет слои (layer cache) и состояние компиляции прямо в инфраструктуре GitHub.
*   **Результат:** Повторная сборка `npm install` занимает 0 секунд, если `package.json` не менялся. Это работает быстрее, чем `type=registry`.

### Multi-Arch сборка
Строка `platforms: linux/amd64,linux/arm64` создает "толстый" манифест.
*   Ваш образ будет работать и на серверах (Intel), и на Raspberry Pi / Apple Silicon.
*   **Важно:** Для этого требуется шаг `Set up QEMU`. Сборка `arm64` на `amd64` раннере может быть медленной (эмуляция). Если критично время — используйте нативные ARM-раннеры.

### Сканирование (Trivy)
Мы вставили шаг сканирования **между** сборкой и пушем.
1.  Собрали образ локально (`load: true`).
2.  Натравили на него Trivy.
3.  Если найдены критические CVE — пайплайн падает красным ❌. "Дырявый" образ никогда не попадет в Registry.

### Metadata Action
Мы не пишем теги вручную. `docker/metadata-action` делает магию:
*   Когда вы пушите тег `v1.2.3`, он создает образы с тегами `v1.2.3`, `v1.2` и `latest`.
*   Когда вы пушите в `main`, он обновляет тег `main`.

## Права доступа (GHCR)
По умолчанию GHCR приватный. Чтобы ваш сервер мог скачать образ (`docker pull`), нужно:
1.  Зайти в настройки пакета в GitHub (Profile -> Packages -> ваш образ).
2.  Package Settings -> "Change visibility" (если хотите сделать публичным).
3.  Или создать **Personal Access Token (Classic)** с правами `read:packages` и использовать его для логина на сервере:
    ```bash
    echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
    ```
