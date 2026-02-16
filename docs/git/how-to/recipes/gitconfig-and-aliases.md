---
title: "Рецепт: .gitconfig и алиасы"
type: how-to
tags: [git, recipe, config, aliases, gitconfig, hooks, workflow]
related:
  - "[[git/reference/cheatsheet]]"
  - "[[git/tutorials/01-basics]]"
  - "[[git/how-to/commit-conventions]]"
---

# Рецепт: .gitconfig и алиасы

> Готовая конфигурация для продуктивной работы с Git.

## ~/.gitconfig

```ini
[user]
    name = Your Name
    email = email@example.com

[init]
    defaultBranch = main

[core]
    editor = vim
    autocrlf = input          # LF на Linux/Mac, CRLF→LF при commit на Windows
    pager = less -FRX

[pull]
    rebase = true              # pull --rebase по умолчанию

[push]
    default = current          # push текущую ветку
    autoSetupRemote = true     # автоматически -u при первом push

[merge]
    conflictStyle = diff3      # показывать base в конфликтах

[diff]
    colorMoved = default

[rerere]
    enabled = true             # запоминать решения конфликтов

[alias]
    s = status -sb
    l = log --oneline -20
    lg = log --oneline --graph --all --decorate
    co = checkout
    sw = switch
    br = branch
    cm = commit -m
    ca = commit --amend --no-edit
    d = diff
    ds = diff --staged
    unstage = restore --staged
    undo = reset --soft HEAD~1
    last = log -1 HEAD --stat
    who = shortlog -sne
    aliases = config --get-regexp alias

[color]
    ui = auto
```

## Полезные алиасы

```bash
# Установить по одному
git config --global alias.s "status -sb"
git config --global alias.l "log --oneline -20"
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.ca "commit --amend --no-edit"
```

### Использование

```bash
git s              # короткий status
git l              # последние 20 коммитов
git lg             # граф веток
git undo           # откатить последний коммит (сохранить изменения)
git ca             # добавить в последний коммит
```

## Git Hooks (автоматизация)

Скрипты в `.git/hooks/`, выполняются автоматически.

### pre-commit (проверка перед коммитом)

```bash
# .git/hooks/pre-commit
#!/bin/bash
# Запретить коммит в main
branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$branch" = "main" ]; then
    echo "❌ Direct commits to main are not allowed. Use a feature branch."
    exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

### commit-msg (проверка сообщения)

```bash
# .git/hooks/commit-msg
#!/bin/bash
# Проверить Conventional Commits формат
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore|ci)(\(.+\))?: .{1,72}"
if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Commit message must follow Conventional Commits:"
    echo "   feat: add user login"
    echo "   fix(auth): resolve null pointer"
    exit 1
fi
```

## Глобальный .gitignore

```bash
git config --global core.excludesFile ~/.gitignore_global
```

```bash
# ~/.gitignore_global
.DS_Store
Thumbs.db
.idea/
.vscode/
*.swp
*~
.env
```