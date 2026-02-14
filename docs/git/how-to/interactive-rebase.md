---
title: "Interactive Rebase"
type: how-to
tags: [git, rebase, interactive, squash, reword, reorder, history]
related:
  - "[[git/how-to/undo-mistakes]]"
  - "[[git/tutorials/02-branching]]"
  - "[[git/explanation/internals]]"
---

# Interactive Rebase

> **TL;DR:** `git rebase -i HEAD~N` — переписать последние N коммитов.
> squash (объединить), reword (переименовать), edit (изменить), drop (удалить).
> ⚠️ Не rebase то, что уже запушено в shared ветку.

## Запуск

```bash
# Последние 3 коммита
git rebase -i HEAD~3

# От конкретного коммита (не включая его)
git rebase -i abc123
```

Откроется редактор:

```
pick abc123 feat: add auth
pick def456 fix: typo
pick 789abc feat: add logout
```

## Операции

| Команда | Что делает |
|---------|-----------|
| `pick` | Оставить как есть |
| `reword` / `r` | Изменить commit message |
| `squash` / `s` | Объединить с предыдущим (сохранить оба сообщения) |
| `fixup` / `f` | Объединить с предыдущим (отбросить сообщение) |
| `edit` / `e` | Остановиться, дать изменить коммит |
| `drop` / `d` | Удалить коммит |

## Примеры

### Объединить 3 коммита в 1 (squash)

```
pick abc123 feat: add auth
squash def456 fix: typo in auth
squash 789abc feat: add logout to auth
```

Результат: один коммит с объединённым сообщением.

### Переименовать коммит

```
reword abc123 feat: add authentication module
pick def456 fix: typo
```

### Изменить порядок

```
pick def456 fix: typo
pick abc123 feat: add auth
```

Просто поменять строки местами.

### Удалить коммит

```
pick abc123 feat: add auth
drop def456 experimental: broken code
pick 789abc feat: add logout
```

## Rebase ветки на main

```bash
git switch feature/auth
git rebase main
# Переносит коммиты feature/auth поверх main

# Если конфликты:
# 1. Решить конфликт в файлах
# 2. git add .
# 3. git rebase --continue

# Отменить rebase
git rebase --abort
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Rebase общей ветки → поломал историю у коллег | **Золотое правило:** не rebase запушенные shared ветки. `git push --force-with-lease` если неизбежно |
| Запутался во время rebase | `git rebase --abort` — вернуться к началу |
| Потерял коммит после rebase | `git reflog` → найти SHA → `git cherry-pick SHA` |