---
title: "02 — Ветки и merge"
type: tutorial
tags: [git, tutorial, branch, merge, conflict, checkout, switch]
related:
  - "[[git/tutorials/01-basics]]"
  - "[[git/tutorials/03-collaboration]]"
  - "[[git/explanation/branching-models]]"
---

# Tutorial 02 — Ветки и merge

> **Цель:** Создавать ветки, переключаться, мержить, решать конфликты.

**Время:** ~20 минут
**Требования:** Пройден Tutorial 01.

## Шаг 1. Создание ветки

```bash
# Посмотреть текущую ветку
git branch
# * main

# Создать и переключиться
git switch -c feature/auth
# или (старый синтаксис)
git checkout -b feature/auth

git branch
#   main
# * feature/auth
```

## Шаг 2. Работа в ветке

```bash
echo "def login(): pass" > auth.py
git add auth.py
git commit -m "feat: add auth module"

echo "def logout(): pass" >> auth.py
git add auth.py
git commit -m "feat: add logout"

git log --oneline
# def456 feat: add logout
# abc123 feat: add auth module
# 111111 Initial commit
```

## Шаг 3. Merge

```bash
# Вернуться в main
git switch main

# Слить feature в main
git merge feature/auth
# Fast-forward (или merge commit)

git log --oneline --graph
# * def456 feat: add logout
# * abc123 feat: add auth module
# * 111111 Initial commit

# Удалить ветку (она вмержена)
git branch -d feature/auth
```

## Шаг 4. Конфликт и его решение

```bash
# Создать конфликт
git switch -c feature-a
echo "version = 1" > config.py
git add . && git commit -m "version 1"

git switch main
echo "version = 2" > config.py
git add . && git commit -m "version 2"

# Мержим — конфликт!
git merge feature-a
# CONFLICT (content): Merge conflict in config.py

# Посмотреть конфликт
cat config.py
# <<<<<<< HEAD
# version = 2
# =======
# version = 1
# >>>>>>> feature-a
```

Решение:
```bash
# Отредактировать файл — оставить нужное, убрать маркеры
echo "version = 2  # merged from both" > config.py

# Отметить как решённый
git add config.py
git commit -m "merge: resolve config conflict"
```

## Шаг 5. Полезные команды

```bash
git branch -a              # все ветки (включая remote)
git branch -v              # с последним коммитом
git branch --merged        # вмерженные в текущую
git branch --no-merged     # не вмерженные

git switch -               # переключиться на предыдущую ветку
git stash                  # спрятать незакоммиченные изменения
git stash pop              # достать обратно
```

## Что мы изучили

| Команда | Что делает |
|---------|-----------|
| `git switch -c name` | Создать и переключиться на ветку |
| `git merge branch` | Слить ветку в текущую |
| `git branch -d name` | Удалить вмерженную ветку |
| `git stash` / `git stash pop` | Спрятать / достать изменения |
| Маркеры конфликта | `<<<<<<<`, `=======`, `>>>>>>>` |

## Что дальше

→ [[git/tutorials/03-collaboration]] — remote, push, pull, pull request
