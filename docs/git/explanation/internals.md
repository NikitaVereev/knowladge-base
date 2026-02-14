---
title: "Как Git устроен внутри"
type: explanation
tags: [git, explanation, internals, objects, blob, tree, commit, hash, dag]
related:
  - "[[git/tutorials/01-basics]]"
  - "[[git/explanation/branching-models]]"
---

# Как Git устроен внутри

> **TL;DR:** Git = направленный ациклический граф (DAG) из объектов.
> 3 типа объектов: blob (файл), tree (директория), commit (снимок).
> Ветка = указатель на commit. HEAD = указатель на текущую ветку.

## Объекты Git

Всё хранится в `.git/objects/` как объекты, идентифицируемые SHA-1 хешем.

```
blob   — содержимое файла (без имени!)
tree   — директория (список blob + tree с именами)
commit — снимок: указатель на tree + author + message + parent commit
tag    — именованный указатель на commit
```

```
commit abc123
├── tree def456
│   ├── blob 111111 → README.md
│   ├── blob 222222 → app.py
│   └── tree 333333 → src/
│       └── blob 444444 → main.py
├── parent: commit 000000
├── author: Name <email>
└── message: "feat: add auth"
```

## Ветки и HEAD

**Ветка** — файл в `.git/refs/heads/`, содержит SHA коммита:

```bash
cat .git/refs/heads/main
# abc123def456...

cat .git/HEAD
# ref: refs/heads/main
```

**HEAD** — указатель на текущую ветку (или коммит в detached state).

Когда вы делаете `git commit`:
1. Создаётся новый commit-объект (parent = текущий)
2. Ветка перемещается на новый commit
3. HEAD остаётся на ветке

## Staging Area (Index)

Промежуточная область между working directory и repository. Файл `.git/index`.

```
Working Dir  →  git add  →  Index (staging)  →  git commit  →  Repository
```

`git add` добавляет blob-объект и обновляет index. `git commit` создаёт tree из index и commit-объект.

## Immutability

Объекты Git **неизменяемы**. «Изменение» коммита (amend, rebase) на самом деле создаёт новый объект. Старый остаётся в `.git/objects/` (пока garbage collection не удалит).

```bash
# Reflog хранит историю перемещений HEAD (30 дней)
git reflog
# abc123 HEAD@{0}: commit: feat: add auth
# 000000 HEAD@{1}: commit (initial): init
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| «Потерял коммит» | `git reflog` → найти SHA → `git checkout SHA` |
| Detached HEAD | `git switch main` (или `git switch -c new-branch` чтобы сохранить) |
| `.git/` удалён | Репозиторий потерян. Восстановить из remote: `git clone` |
| Огромный `.git/` | `git gc` для сборки мусора. `git filter-repo` для удаления больших файлов из истории |
