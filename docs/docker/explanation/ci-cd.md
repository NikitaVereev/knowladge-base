---
title: Контейнеры в CI/CD
type: explanation
tags: [docker, cicd, pipeline, build, test, promote, testcontainers, github-actions]
sources:
  docs: "https://docs.docker.com/build/ci/"
related:
  - "[[docker/how-to/github-actions-pipeline]]"
  - "[[docker/explanation/registry]]"
  - "[[docker/explanation/images-and-layers]]"
---

# Контейнеры в CI/CD (The Build Pipeline)

Внедрение Docker кардинально меняет процесс непрерывной интеграции и доставки (CI/CD). Контейнер становится **единым артефактом**, который проходит путь от коммита разработчика до продакшена. Мы больше не передаем zip-архивы с кодом и инструкции по настройке сервера.

## Традиционный vs Docker-based пайплайн

### Традиционный подход (Mutable)
1.  **Build Server:** На агенте (Jenkins) должны стоять: Java 17, Maven 3.8, Node.js 16.
2.  **Artifact:** JAR-файл или папка с кодом.
3.  **Deploy:** Ansible-скрипт копирует файл на сервер, правит конфиги, перезапускает systemd.
4.  **Проблема:** "It works on my machine". Версии Java на Jenkins, Dev-сервере и Prod-сервере могут отличаться.

### Docker-based подход (Immutable)
1.  **Build Server:** На агенте стоит **только Docker**.
2.  **Build:** Сборка происходит *внутри* контейнера (`docker build`). Все компиляторы изолированы.
3.  **Artifact:** Docker Image с конкретным дайджестом.
4.  **Deploy:** `docker pull` && `docker run`.

## Этапы Docker-пайплайна

### 1. Build (Сборка)
На этом этапе код превращается в образ.
*   **Сборка в контейнере:** Используйте Multi-stage build. Первый этап (`FROM maven`) компилирует, второй (`FROM openjdk`) — запускает. Это избавляет CI-агентов от необходимости иметь установленные SDK.
*   **Кэширование:** Использование `--cache-from` или `BuildKit` (`docker buildx`) позволяет переиспользовать слои с зависимостями (`npm install`), ускоряя сборку в разы.

### 2. Test (Тестирование)
Тесты запускаются не на хосте CI, а внутри контейнеров.
*   **Unit Tests:** Запускаются во время сборки (`RUN npm test`). Если тесты упали — образ не собирается.
*   **Integration Tests:**
    *   **Docker Compose:** CI поднимает полное окружение (App + Postgres + Redis) через `docker compose up -d`.
    *   **Testcontainers:** Библиотека (Java, Go, Node, Python), позволяющая запускать одноразовые контейнеры (БД, брокеры сообщений) прямо из кода тестов.

### 3. Scan (Сканирование)
Перед пушем в реестр образ проверяется на уязвимости (Static Analysis).
*   Инструменты (Trivy, Docker Scout) анализируют список пакетов ОС и зависимостей приложения на наличие известных CVE.
*   Пайплайн блокируется, если найдены уязвимости уровня High/Critical.

### 4. Push & Promote (Публикация)
*   Образ пушится в Registry с уникальным тегом (например, `myapp:commit-sha`).
*   **Promotion Pattern:** Мы **не пересобираем** образ для продакшена. Мы берем тот же самый дайджест, который прошел тесты в Stage, и добавляем ему тег `release-v1.0`. Это гарантирует, что в проде работает *бинарно тот же самый* код, что был протестирован.

## Проблема "Docker-in-Docker" (DinD)

Как запускать `docker build` внутри CI-агента, который сам запущен в Docker-контейнере (например, GitLab Runner)?

1.  **Docker-in-Docker (DinD):** Запуск демона Docker внутри контейнера (режим `--privileged`).
    *   *Минусы:* Медленно, проблемы с файловой системой (UnionFS поверх UnionFS), небезопасно.
2.  **Docker-out-of-Docker (DooD):** Проброс сокета хоста (`-v /var/run/docker.sock:/var/run/docker.sock`).
    *   *Плюсы:* Агент использует Docker демона хоста. Быстрый кэш.
    *   *Минусы:* Проблемы с именами контейнеров (конфликты на хосте), агенты могут "убить" друг друга или хост.
3.  **Kaniko / Buildah:** Инструменты для сборки образов без демона Docker и без привилегий root. Стандарт для Kubernetes-среды (Tekton, Jenkins X).

## Пример идеального Workflow

1.  **Git Push:** Разработчик пушит код.
2.  **CI Build:** Запускается `docker build`. Выполняются Unit-тесты. Создается образ-кандидат.
3.  **CI Scan:** Trivy проверяет образ.
4.  **CI Push:** Образ улетает в Registry с тегом `myapp:sha-123`.
5.  **CD Stage:** ArgoCD/Compose обновляет стейдж-сервер на версию `sha-123`.
6.  **E2E Tests:** Запускаются Cypress-тесты против стейджа.
7.  **CD Prod:** При аппруве, образу `sha-123` ставится тег `v1.0`, и он деплоится в прод.