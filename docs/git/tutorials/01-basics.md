---
title: "01 — Основы Git"
type: tutorial
tags: [git, tutorial, init, add, commit, log, diff, status]
related:
  - "[[git/tutorials/02-branching]]"
  - "[[git/explanation/internals]]"
  - "[[git/reference/cheatsheet]]"
---

# Tutorial 01 — Основы Git

> **Цель:** init, add, commit, status, log, diff. Понять staging area.

**Время:** ~20 минут
**Требования:** Git установлен (`git --version`).

## Шаг 1. Настройка (один раз)

```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global init.defaultBranch main
git config --global core.editor vim          # или code, nano
```

## Шаг 2. Создание репозитория

```bash
mkdir myproject && cd myproject
git init
# Initialized empty Git repository in /home/user/myproject/.git/

ls -la .git/
# config  HEAD  hooks/  objects/  refs/
```

## Шаг 3. Первый коммит

```bash
# Создать файл
echo "# My Project" > README.md

# Статус — что изменилось?
git status
# Untracked files: README.md

# Добавить в staging area
git add README.md

git status
# Changes to be committed: new file: README.md

# Зафиксировать
git commit -m "Initial commit: add README"
```

## Шаг 4. Staging Area (индекс)

```
Working Directory    →  Staging Area  →  Repository
    (файлы)            (git add)        (git commit)
```

```bash
# Изменить файл
echo "Description" >> README.md
echo "print('hello')" > app.py

git status
# Changes not staged: README.md
# Untracked: app.py

# Добавить всё
git add .

# Или выборочно
git add README.md

# Посмотреть что в staging
git diff --staged
```

## Шаг 5. История

```bash
git log
# commit abc123 (HEAD -> main)
# Author: Your Name
# Date: ...
# Initial commit: add README

# Компактный формат
git log --oneline
# abc123 Initial commit: add README

# С графом веток
git log --oneline --graph --all

# Что изменилось в коммите
git show abc123
```

## Шаг 6. Diff — что изменилось

```bash
# Рабочая директория vs staging
git diff

# Staging vs последний коммит
git diff --staged

# Между коммитами
git diff abc123 def456

# Конкретный файл
git diff README.md
```

## Шаг 7. .gitignore

```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.pyc
.venv/

# Node
node_modules/
dist/

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db

# Environment
.env
*.log
EOF

git add .gitignore
git commit -m "Add .gitignore"
```

## Что мы изучили

| Команда | Что делает |
|---------|-----------|
| `git init` | Создать репозиторий |
| `git add .` | Добавить всё в staging |
| `git commit -m "msg"` | Зафиксировать изменения |
| `git status` | Что изменилось |
| `git log --oneline` | История коммитов |
| `git diff` | Различия в файлах |

## Что дальше

→ [[git/tutorials/02-branching]] — ветки, merge, конфликты