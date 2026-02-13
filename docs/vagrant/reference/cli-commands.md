---
title: "Справочник: Vagrant CLI"
type: reference
tags: [vagrant, reference, cli, commands]
sources:
  original: "devops/vagrant/reference/cli-commands.md"
related:
  - "[[vagrant/tutorials/01-first-vm]]"
  - "[[vagrant/how-to/configure-vagrantfile]]"
---

# Справочник: Vagrant CLI

## Жизненный цикл

| Команда | Описание |
|---------|---------|
| `vagrant init [box]` | Создать Vagrantfile |
| `vagrant up [name]` | Создать и запустить VM |
| `vagrant halt [name]` | Выключить (ACPI shutdown) |
| `vagrant suspend [name]` | Пауза (сохранить RAM на диск) |
| `vagrant resume [name]` | Вывести из паузы |
| `vagrant reload [name]` | Перезагрузить (применить изменения Vagrantfile) |
| `vagrant destroy [-f] [name]` | Удалить VM и диски |

## Работа с машиной

| Команда | Описание |
|---------|---------|
| `vagrant ssh [name]` | Подключиться по SSH |
| `vagrant ssh-config [name]` | Показать SSH-параметры (IP, порт, ключ) |
| `vagrant provision [name]` | Запустить провижинеры |
| `vagrant reload --provision` | Перезагрузка + provisioning |
| `vagrant port [name]` | Проброшенные порты |
| `vagrant status` | Статус текущего проекта |
| `vagrant global-status` | Статус всех VM на хосте |
| `vagrant global-status --prune` | Очистить устаревшие записи |

## Управление образами (Boxes)

| Команда | Описание |
|---------|---------|
| `vagrant box list` | Скачанные боксы |
| `vagrant box add [name]` | Скачать бокс |
| `vagrant box update` | Обновить бокс |
| `vagrant box prune` | Удалить старые версии |
| `vagrant box remove [name]` | Удалить бокс |

## Снэпшоты

| Команда | Описание |
|---------|---------|
| `vagrant snapshot save [name] SNAP` | Сохранить snapshot |
| `vagrant snapshot restore [name] SNAP` | Восстановить |
| `vagrant snapshot list` | Список снэпшотов |
| `vagrant snapshot delete SNAP` | Удалить |

## Полезные флаги

| Флаг | Описание |
|------|---------|
| `--provision` | Принудительно запустить provisioning |
| `--no-provision` | Пропустить provisioning |
| `-f` | Без подтверждения (для destroy) |
| `--parallel` | Параллельный запуск нескольких VM |