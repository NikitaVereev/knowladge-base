---
title: "Реестры образов Docker"
description: "Обзор Docker Hub, GitHub Container Registry и локальных реестров. Где хранить и как распространять образы."
---


Registry — это сервер хранения и распространения Docker-образов. Он позволяет команде или сообществу обмениваться готовыми образами.

## Типы реестров

### Docker Hub (Публичный реестр)
Официальный реестр Docker. Миллионы готовых образов (nginx, postgres, node и др.).
* **Бесплатный план:** Неограниченное количество публичных репозиториев + 1 приватный.
* **Ограничения:** 100 pull-запросов каждые 6 часов для анонимных пользователей.
* **Форма образа:** `username/image:tag` (например, `johndoe/webapp:1.0`)

### GitHub Container Registry (ghcr.io)
Встроенный реестр GitHub для пакетов.
* **Плюсы:** Тесная интеграция с GitHub Actions, приватные образы бесплатно (в рамках квот).
* **Форма образа:** `ghcr.io/username/image:tag`

### Приватный реестр (Self-hosted)
Собственный сервер на базе `registry:2` образа.
* **Когда нужен:** Конфиденциальность, автономная работа, нестандартные требования.

## Сравнение форматов тегов

| Реестр | Формат тега | Пример |
|--------|-------------|--------|
| Docker Hub | `username/repo:tag` | `alice/api:v1.0` |
| GitHub | `ghcr.io/username/repo:tag` | `ghcr.io/alice/api:v1.0` |
| GitLab | `registry.gitlab.com/user/repo:tag` | `registry.gitlab.com/alice/api:v1.0` |
| Private | `registry.domain.com/repo:tag` | `reg.company.com/api:v1.0` |

## Связанные материалы

- [[devops/docker/how-to/publish-images|Публикация образов в реестры]]
- [[devops/docker/reference/versioning-strategy|Стратегии тегирования и версионирование]]
