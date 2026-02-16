---
title: "Git"
type: index
tags: [git, vcs, version-control, github, branching]
---

# Git

Распределённая система контроля версий. Отслеживает изменения, ветвление, совместная работа, история.

## Explanation

| Документ | Описание |
|----------|----------|
| [[git/explanation/internals]] | Объекты (blob, tree, commit), HEAD, staging area, immutability |
| [[git/explanation/branching-models]] | GitHub Flow, Trunk-Based, Git Flow, Conventional Commits |

## Tutorials

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[git/tutorials/01-basics]] | init, add, commit, status, log, diff, .gitignore |
| 02 | [[git/tutorials/02-branching]] | branch, switch, merge, конфликты, stash |
| 03 | [[git/tutorials/03-collaboration]] | remote, clone, push, pull, fork, Pull Request |

## How-to

| Документ | Описание |
|----------|----------|
| [[git/how-to/interactive-rebase]] | squash, reword, reorder, edit, drop коммитов |
| [[git/how-to/undo-mistakes]] | amend, reset, revert, restore, reflog, cherry-pick |
| [[git/how-to/commit-conventions]] | Conventional Commits, commitlint + husky, автоматический CHANGELOG |

## Recipes

| Рецепт | Описание |
|--------|----------|
| [[git/how-to/recipes/gitconfig-and-aliases]] | .gitconfig, алиасы, hooks, глобальный .gitignore |

## Reference

| Документ | Описание |
|----------|----------|
| [[git/reference/cheatsheet]] | Все команды: init, branch, merge, remote, undo, stash, tag |

## Быстрый старт

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin git@github.com:user/repo.git
git push -u origin main
```

## Связанные разделы

- [[ssh/index]] — SSH (аутентификация для Git remote)