---
title: "03 — Совместная работа"
type: tutorial
tags: [git, tutorial, remote, push, pull, clone, fork, pull-request, github]
related:
  - "[[git/tutorials/02-branching]]"
  - "[[git/explanation/branching-models]]"
  - "[[ssh/tutorials/01-getting-started]]"
---

# Tutorial 03 — Совместная работа

> **Цель:** remote-репозитории, clone, push, pull, fork, pull request.

**Время:** ~20 минут
**Требования:** Пройден Tutorial 02. Аккаунт GitHub/GitLab. SSH-ключ настроен.

## Шаг 1. Remote-репозиторий

```bash
# Добавить remote
git remote add origin git@github.com:user/repo.git

# Посмотреть remotes
git remote -v
# origin  git@github.com:user/repo.git (fetch)
# origin  git@github.com:user/repo.git (push)
```

## Шаг 2. Push (отправить)

```bash
# Первый push (установить upstream)
git push -u origin main

# Последующие
git push
```

## Шаг 3. Clone (скачать)

```bash
git clone git@github.com:user/repo.git
cd repo
git log --oneline
```

## Шаг 4. Pull (получить изменения)

```bash
git pull
# = git fetch + git merge

# Только скачать (без merge)
git fetch
git log origin/main --oneline
git merge origin/main
```

## Шаг 5. Workflow с ветками

```bash
# 1. Создать ветку
git switch -c feature/payment

# 2. Работать, коммитить
git add . && git commit -m "feat: add payment"

# 3. Запушить ветку
git push -u origin feature/payment

# 4. Создать Pull Request (в GitHub/GitLab UI)

# 5. После code review — merge в main (через UI)

# 6. Локально обновить main
git switch main
git pull
git branch -d feature/payment
```

## Шаг 6. Fork workflow (open source)

```bash
# 1. Fork репозитория (в GitHub UI)

# 2. Клонировать свой fork
git clone git@github.com:YOUR-USER/repo.git

# 3. Добавить upstream (оригинал)
git remote add upstream git@github.com:ORIGINAL-USER/repo.git

# 4. Создать ветку, работать, запушить
git switch -c fix/typo
git add . && git commit -m "fix: typo in readme"
git push -u origin fix/typo

# 5. Создать Pull Request (из вашего fork в оригинал)

# 6. Синхронизировать с upstream
git switch main
git fetch upstream
git merge upstream/main
git push
```

## Что мы изучили

| Команда | Что делает |
|---------|-----------|
| `git remote add origin URL` | Связать с remote |
| `git push -u origin main` | Первый push |
| `git pull` | Получить + merge |
| `git clone URL` | Скачать репозиторий |
| `git fetch` | Скачать без merge |

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `rejected: non-fast-forward` | `git pull --rebase` перед push |
| `Permission denied (publickey)` | SSH-ключ не настроен. См. [[ssh/tutorials/01-getting-started]] |
| Push в чужой репозиторий | Нужен fork + pull request |
| Конфликт при pull | Решить как обычный merge conflict |
