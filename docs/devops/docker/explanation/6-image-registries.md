---
title: "6 Реестры образов Docker"
description: "Что такое Registry, обзор Docker Hub, GitHub Container Registry (GHCR) и self-hosted решений. Как хранить и распространять образы."
---

**Container Registry** (реестр контейнеров) — это серверное приложение для хранения и распространения Docker-образов. Он работает как хранилище артефактов: CI/CD система собирает образ и делает `push` в реестр, а серверы делают `pull`, чтобы запустить приложение. 

## Типы реестров

### 1. Docker Hub (Публичный стандарт)
Официальный и самый большой реестр. Если вы пишете `FROM node`, Docker ищет этот образ именно здесь. 

- **Бесплатный план:** Неограниченные публичные репозитории.
- **Ограничения (Rate Limits):** Для анонимных пользователей — 100 pull-запросов за 6 часов, для авторизованных (бесплатных) — 200. Это часто ломает CI/CD пайплайны. 
- **Формат имени:** `username/image:tag` (например, `myname/myapp:v1`). Если username нет (например, `nginx`), это официальный образ от вендора.

### 2. GitHub Container Registry (ghcr.io)
Часть экосистемы GitHub Packages. Становится стандартом для open-source и коммерческих проектов, живущих на GitHub. 

- **Плюсы:** Бесшовная авторизация через `GITHUB_TOKEN` в GitHub Actions. Не нужно создавать отдельные секреты для доступа к образам внутри пайплайна.
- **Цена:** Бесплатно для публичных образов. Для приватных — входит в квоты GitHub (Storage + Data Transfer).
- **Формат имени:** `ghcr.io/owner/image:tag` (нужно указывать полный путь).

### 3. Cloud Registries (AWS ECR, Google GCR/Artifact Registry, Yandex CR)
Реестры от облачных провайдеров.
- **Когда использовать:** Если ваша инфраструктура уже в облаке. Например, Kubernetes в AWS (EKS) будет скачивать образы из AWS ECR намного быстрее и дешевле (трафик внутри дата-центра), чем из Docker Hub.

### 4. Self-hosted (Private Registry)
Собственный инстанс реестра внутри периметра компании. Часто запускается как контейнер `registry:2` или через Harbor (Enterprise-решение с GUI, сканированием уязвимостей и RBAC). 
- **Когда нужен:** Строгие требования безопасности (air-gapped контур без интернета), экономия трафика, полный контроль над данными.

## Сравнение форматов адресации

Docker определяет, куда стучаться, по первой части тега (домена). Если домена нет, подразумевается `docker.io` (Docker Hub).

| Реестр | Полный формат образа | Пример |
| :--- | :--- | :--- |
| **Docker Hub** | `(docker.io/)username/repo:tag` | `alice/web-api:1.0` |
| **GitHub (GHCR)** | `ghcr.io/username/repo:tag` | `ghcr.io/alice/web-api:1.0` |
| **GitLab Registry** | `registry.gitlab.com/group/project/img:tag` | `registry.gitlab.com/corp/backend/api:main` |
| **AWS ECR** | `account-id.dkr.ecr.region.amazonaws.com/repo:tag` | `123456789.dkr.ecr.us-east-1.amazonaws.com/api:v1` |
| **Localhost** | `localhost:5000/repo:tag` | `localhost:5000/my-test:latest` |

## Связанные материалы

- [[8-publish-images|Публикация образов в реестры]]
- [[5-versioning-strategy|Стратегии тегирования и версионирование]]
