---
title: "Docker"
description: "Контейнеризация: образы, Dockerfile, Compose, сети, хранение, безопасность, CI/CD, Swarm"
tags: [docker, containers, devops]
---

# Docker

## Я хочу...

| Задача | Куда идти |
|--------|-----------|
| **Выучить Docker с нуля** | [[docker/tutorials/01-first-container]] → читай по порядку |
| **Dockerfile для Next.js** | [[docker/how-to/recipes/nextjs]] |
| **Dockerfile для Node.js** | [[docker/how-to/recipes/nodejs]] |
| **Dockerfile для Python** | [[docker/how-to/recipes/python-fastapi]] / [[docker/how-to/recipes/python-django]] |
| **Dockerfile для Go** | [[docker/how-to/recipes/go]] |
| **Полный стек (App + DB + Redis + Nginx)** | [[docker/how-to/recipes/fullstack]] |
| **Понять как работает Docker** | [[docker/explanation/architecture]] |
| **Понять сети Docker** | [[docker/explanation/networking]] |
| **Настроить CI/CD** | [[docker/how-to/github-actions-pipeline]] |
| **Подготовить к production** | [[docker/how-to/production-hardening]] |
| **Отладить контейнер** | [[docker/how-to/debug-containers]] |
| **Найти команду** | [[docker/reference/cheatsheet]] |
| **Docker Swarm** | [[docker/explanation/swarm]] — концепции, [[docker/how-to/swarm/deploy-stack]] — практика |

---

## Tutorials — путь обучения

Читай по порядку. После прохождения всех уроков — уверенно работаешь с Docker.

| # | Урок | Что освоишь |
|---|------|-------------|
| 01 | [[docker/tutorials/01-first-container]] | Запуск, порты, логи, жизненный цикл контейнера |
| 02 | [[docker/tutorials/02-building-images]] | Dockerfile: от наивного к production-ready (multi-stage) |
| 03 | [[docker/tutorials/03-compose-app]] | Docker Compose: App + DB + Redis, healthcheck, depends_on |
| 04 | [[docker/tutorials/04-networking-workshop]] | Сети на практике: bridge, DNS, связь контейнеров |
| 05 | [[docker/tutorials/05-debugging-workshop]] | Сломанные контейнеры — находим и чиним |
| 06 | [[docker/tutorials/06-swarm-cluster]] | Кластер Swarm на Vagrant: init, join, deploy |

## Explanation — как устроено

Не привязаны к порядку. Каждый файл — самостоятельная тема. Читай когда хочешь разобраться «почему».

| Тема | Описание |
|------|----------|
| [[docker/explanation/architecture]] | Client → Daemon → containerd → runc, OCI, namespaces, cgroups |
| [[docker/explanation/images-and-layers]] | UnionFS, OverlayFS, Copy-on-Write, base images |
| [[docker/explanation/storage]] | Volumes vs Bind Mounts vs tmpfs, персистентность данных |
| [[docker/explanation/networking]] | CNM, bridge/host/overlay, veth, iptables, DNS |
| [[docker/explanation/configuration]] | ENV vs ARG, secrets, 12-Factor, передача конфигурации |
| [[docker/explanation/compose-model]] | Декларативный подход, проекты, зависимости, профили |
| [[docker/explanation/registry]] | OCI Distribution, манифесты, multi-arch, теги vs дайджесты |
| [[docker/explanation/runtime-behavior]] | Signals, PID 1, exit codes, restart policies, logging |
| [[docker/explanation/security]] | Capabilities, seccomp, AppArmor, rootless, supply chain |
| [[docker/explanation/ci-cd]] | Build → Test → Scan → Push, DinD vs DooD, Kaniko |
| [[docker/explanation/swarm]] | Архитектура кластера, Raft, overlay, service discovery |

## How-to — решение задач

Иди сюда когда нужно сделать конкретную вещь.

| Задача | Файл |
|--------|------|
| Установить Docker | [[docker/how-to/install]] |
| Оптимизировать сборку | [[docker/how-to/optimize-builds]] |
| Отладить контейнер | [[docker/how-to/debug-containers]] |
| Управлять ресурсами (CPU/RAM) | [[docker/how-to/manage-resources]] |
| Очистка и обслуживание | [[docker/how-to/cleanup-maintenance]] |
| Локальная сеть и SSH | [[docker/how-to/local-networking]] |
| CI/CD через GitHub Actions | [[docker/how-to/github-actions-pipeline]] |
| Работа с Private Registry | [[docker/how-to/private-registry]] |
| Production hardening | [[docker/how-to/production-hardening]] |

### Swarm

| Задача | Файл |
|--------|------|
| Развернуть стек | [[docker/how-to/swarm/deploy-stack]] |
| Управление секретами | [[docker/how-to/swarm/manage-secrets]] |
| Настройка сетей | [[docker/how-to/swarm/configure-networks]] |
| Размещение сервисов | [[docker/how-to/swarm/manage-placement]] |
| Stateful-приложения (БД) | [[docker/how-to/swarm/deploy-stateful]] |
| Healthchecks | [[docker/how-to/swarm/configure-healthchecks]] |
| Бэкап и восстановление | [[docker/how-to/swarm/backup-restore]] |
| Мониторинг кластера | [[docker/how-to/swarm/monitor-cluster]] |
| Отладка сети | [[docker/how-to/swarm/debug-networking]] |

### Recipes — готовые конфиги

Копируй → работает. Каждый рецепт: Dockerfile + compose.yaml + .dockerignore + объяснение.

| Стек | Файл |
|------|------|
| Next.js | [[docker/how-to/recipes/nextjs]] |
| Node.js (Express / Fastify / Nest) | [[docker/how-to/recipes/nodejs]] |
| Python + Django | [[docker/how-to/recipes/python-django]] |
| Python + FastAPI | [[docker/how-to/recipes/python-fastapi]] |
| Go | [[docker/how-to/recipes/go]] |
| Java + Spring Boot | [[docker/how-to/recipes/java-spring]] |
| Nginx (reverse proxy + SSL) | [[docker/how-to/recipes/nginx-reverse-proxy]] |
| PostgreSQL | [[docker/how-to/recipes/postgres]] |
| Redis | [[docker/how-to/recipes/redis]] |
| Fullstack (App + DB + Redis + Nginx) | [[docker/how-to/recipes/fullstack]] |
| Monitoring (Prometheus + Grafana) | [[docker/how-to/recipes/monitoring]] |

## Reference — справочник

| Справочник | Описание |
|------------|----------|
| [[docker/reference/cheatsheet]] | Все команды Docker CLI на одной странице |
| [[docker/reference/dockerfile]] | Все инструкции Dockerfile + best practices |
| [[docker/reference/compose-spec]] | Спецификация compose.yaml |
| [[docker/reference/security-checklist]] | Чек-лист перед production-деплоем |
| [[docker/reference/swarm-cheatsheet]] | Команды Swarm CLI + best practices |
