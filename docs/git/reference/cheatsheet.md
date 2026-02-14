---
title: "Справочник: Git команды"
type: reference
tags: [git, reference, cheatsheet, commands]
related:
  - "[[git/tutorials/01-basics]]"
  - "[[git/how-to/undo-mistakes]]"
  - "[[git/how-to/interactive-rebase]]"
---

# Справочник: Git команды

## Настройка

```bash
git config --global user.name "Name"
git config --global user.email "email"
git config --global init.defaultBranch main
git config --list                  # все настройки
```

## Создание

```bash
git init                           # новый репозиторий
git clone URL                      # скачать существующий
git clone URL dir                  # в конкретную папку
git clone --depth 1 URL            # только последний коммит (быстро)
```

## Базовые операции

```bash
git status                         # что изменилось
git status -sb                     # короткий формат
git add file                       # добавить в staging
git add .                          # добавить всё
git add -p                         # интерактивно (по кускам)
git commit -m "message"            # зафиксировать
git commit --amend                 # исправить последний коммит
```

## Ветки

```bash
git branch                         # список
git branch -a                      # + remote
git branch -v                      # с последним коммитом
git switch -c name                 # создать и переключиться
git switch name                    # переключиться
git switch -                       # предыдущая ветка
git branch -d name                 # удалить (вмерженную)
git branch -D name                 # удалить (любую)
```

## Merge и Rebase

```bash
git merge branch                   # слить ветку
git merge --abort                  # отменить merge
git rebase main                    # перебазировать на main
git rebase -i HEAD~3               # интерактивный rebase
git rebase --abort                 # отменить rebase
git cherry-pick SHA                # скопировать коммит
```

## Remote

```bash
git remote -v                      # список remotes
git remote add origin URL          # добавить
git push -u origin main            # первый push
git push                           # отправить
git push --force-with-lease        # force push (безопасный)
git pull                           # получить + merge
git pull --rebase                  # получить + rebase
git fetch                          # получить без merge
```

## История

```bash
git log --oneline                  # компактный лог
git log --oneline --graph --all    # граф
git log -p file                    # изменения файла
git log --author="Name"            # по автору
git log --since="2 weeks ago"      # по дате
git show SHA                       # детали коммита
git blame file                     # кто менял каждую строку
```

## Отмена

```bash
git restore file                   # отменить изменения в файле
git restore --staged file          # убрать из staging
git reset --soft HEAD~1            # откатить коммит (сохранить staging)
git reset HEAD~1                   # откатить коммит (сохранить файлы)
git reset --hard HEAD~1            # откатить коммит (удалить всё)
git revert SHA                     # отменить коммитом (безопасно)
git reflog                         # история всех действий
```

## Stash

```bash
git stash                          # спрятать изменения
git stash pop                      # достать обратно
git stash list                     # список
git stash drop                     # удалить последний
git stash clear                    # удалить все
```

## Diff

```bash
git diff                           # working dir vs staging
git diff --staged                  # staging vs commit
git diff main..feature             # между ветками
git diff SHA1 SHA2                 # между коммитами
git diff --stat                    # только статистика
```

## Теги

```bash
git tag v1.0.0                     # лёгкий тег
git tag -a v1.0.0 -m "Release"    # аннотированный
git push --tags                    # отправить теги
git tag -d v1.0.0                  # удалить локально
```

## Очистка

```bash
git clean -fd                      # удалить untracked файлы и директории
git clean -fdn                     # dry-run (показать что удалится)
git gc                             # garbage collection
```
