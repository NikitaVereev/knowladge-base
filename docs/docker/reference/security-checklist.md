---
title: "Чек-лист безопасности Docker"
type: reference
tags: [docker, security, hardening, checklist, production]
sources:
  docs: "https://docs.docker.com/engine/security/"
related:
  - "[[docker/explanation/security]]"
  - "[[docker/how-to/production-hardening]]"
  - "[[docker/explanation/architecture]]"
---

# Чек-лист безопасности Docker

> **Справочник:** Чек-лист из 4 категорий. Пройди все пункты перед production-деплоем.
> Каждый чекбокс — конкретное действие.

Этот список проверок необходимо выполнить перед развертыванием контейнеризированного приложения в публичной среде (Production).

## 1. Host & Daemon Hardening

- [ ] **Docker Daemon обновлен** до последней стабильной версии (CVE фиксятся быстро).
- [ ] **Сокет защищен:** Файл `/var/run/docker.sock` имеет права `660` и принадлежит `root:docker`. Доступ к группе `docker` имеют только доверенные пользователи.
- [ ] **Auditd настроен:** Включен аудит для `/var/lib/docker` и конфигурационных файлов демона.
- [ ] **Daemon configuration:** В `/etc/docker/daemon.json` настроены:
    - [ ] `"userns-remap": "default"` (или используется Rootless mode).
    - [ ] `"no-new-privileges": true` (запрет повышения привилегий через setuid бинарники).
    - [ ] `"icc": false` (Inter-Container Communication отключен в дефолтном bridge, если он используется).
    - [ ] `"log-driver"` настроен с ротацией логов.

## 2. Image Security

- [ ] **Minimal Base Image:** Используется `alpine`, `slim` или `distroless`. В образе нет компиляторов (gcc, make) и отладочных утилит.
- [ ] **Фиксированные теги:** `FROM node:18.16.0-alpine`, а не `FROM node:latest`. Лучше всего использовать дайджест (`@sha256:...`).
- [ ] **Trusted Sources:** Образы берутся только из Docker Hub Official Images или доверенного приватного Registry.
- [ ] **Vulnerability Scanning:** Образ просканирован (Trivy, Grype, Snyk) и не имеет критических (Critical/High) уязвимостей.
- [ ] **No Secrets:** В истории слоев (`docker history`) нет паролей, ключей SSH или токенов API.
- [ ] **Verified Artifacts:** Включен `DOCKER_CONTENT_TRUST=1`, образы подписаны.

## 3. Container Runtime

- [ ] **Non-root User:** Приложение запущено от UID > 0 (`USER 1000` в Dockerfile).
- [ ] **Read-Only Rootfs:** Корневая файловая система смонтирована только для чтения (`--read-only`), запись разрешена только в конкретные тома/tmpfs.
- [ ] **No Privileged Mode:** Флаг `--privileged` **никогда** не используется.
- [ ] **Drop Capabilities:** Все capabilities сброшены (`--cap-drop=ALL`), добавлены только необходимые (или ни одной).
- [ ] **Resource Limits:** Жестко заданы лимиты `--memory` и `--cpus`.
- [ ] **Healthchecks:** Настроен `HEALTHCHECK` для предотвращения работы "зомби"-контейнеров.
- [ ] **Restart Policy:** Установлено `--restart on-failure` или `always` для предотвращения DoS через постоянные перезапуски (CrashLoopBackOff).

## 4. Network Security

- [ ] **User-Defined Networks:** Не используется дефолтный `bridge`. Приложение и БД находятся в изолированной сети.
- [ ] **Expose Minimal Ports:** На хост проброшены только необходимые порты (например, 80/443). Порты базы данных (5432) не опубликованы (`-p`), если доступ к ним нужен только внутри сети Docker.
- [ ] **Bind to Specific Interface:** Порты привязаны к конкретному IP, а не ко всем (`-p 127.0.0.1:8080:80` вместо `-p 8080:80`), если сервис не должен быть публичным.