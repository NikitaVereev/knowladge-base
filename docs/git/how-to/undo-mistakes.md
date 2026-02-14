---
title: "Отмена и исправление ошибок"
type: how-to
tags: [git, undo, reset, revert, amend, checkout, restore, reflog]
related:
  - "[[git/how-to/interactive-rebase]]"
  - "[[git/explanation/internals]]"
  - "[[git/reference/cheatsheet]]"
---

# Отмена и исправление ошибок

> **TL;DR:** `amend` — исправить последний коммит. `reset` — откатить (только локально).
> `revert` — отменить коммит новым коммитом (безопасно для shared). `reflog` — спасательный круг.

## Исправить последний коммит

```bash
# Изменить сообщение
git commit --amend -m "feat: correct message"

# Добавить забытый файл
git add forgotten-file.py
git commit --amend --no-edit
```

> ⚠️ `amend` переписывает коммит. Не делать если уже запушен.

## Убрать файл из staging

```bash
git restore --staged file.txt
# или (старый синтаксис)
git reset HEAD file.txt
```

## Отменить изменения в файле

```bash
# Вернуть файл к последнему коммиту
git restore file.txt
# или
git checkout -- file.txt

# ⚠️ Изменения будут ПОТЕРЯНЫ (нет undo)
```

## Reset (откатить коммиты)

```bash
# Soft: коммиты убраны, изменения остаются в staging
git reset --soft HEAD~1

# Mixed (default): коммиты убраны, изменения в working dir
git reset HEAD~1

# Hard: коммиты убраны, изменения УДАЛЕНЫ
git reset --hard HEAD~1
# ⚠️ Данные потеряны!

# Откатить к конкретному коммиту
git reset --hard abc123
```

| Режим | Коммит | Staging | Working Dir |
|-------|--------|---------|------------|
| `--soft` | ❌ Убран | ✅ Сохранён | ✅ Сохранён |
| `--mixed` | ❌ Убран | ❌ Убран | ✅ Сохранён |
| `--hard` | ❌ Убран | ❌ Убран | ❌ Убран |

## Revert (безопасная отмена)

```bash
# Создаёт новый коммит, отменяющий изменения
git revert abc123

# Отменить последний коммит
git revert HEAD

# Отменить несколько коммитов
git revert HEAD~3..HEAD
```

**Revert vs Reset:** revert безопасен для shared веток (не переписывает историю). Reset — только для локальных.

## Reflog — спасательный круг

```bash
# Показать все действия за 30 дней
git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~1
# def456 HEAD@{1}: commit: feat: important feature  ← вот он!

# Восстановить
git reset --hard def456
# или
git cherry-pick def456
```

## Cherry-pick (перенести коммит)

```bash
# Скопировать коммит из другой ветки
git cherry-pick abc123

# Несколько
git cherry-pick abc123 def456

# Без автокоммита (только изменения)
git cherry-pick --no-commit abc123
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `git reset --hard` потерял работу | `git reflog` → найти SHA → `git reset --hard SHA` |
| Запушил секрет (пароль, ключ) | `git filter-repo` для удаления из истории. **Сменить секрет!** |
| Не тот файл в коммите | `git reset --soft HEAD~1` → исправить → commit заново |
| Revert мерж-коммита | `git revert -m 1 MERGE_SHA` (указать parent) |