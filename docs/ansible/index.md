---
title: "Ansible"
type: index
tags: [ansible, automation, configuration-management, infrastructure-as-code]
---

# Ansible

Автоматизация настройки серверов и деплоя приложений. Безагентный (SSH + Python), декларативный (YAML), идемпотентный.

## Explanation (Концепции)

| Документ | Описание |
|----------|----------|
| [[ansible/explanation/architecture]] | Что такое Ansible, push-модель, компоненты, процесс выполнения |
| [[ansible/explanation/inventory]] | Inventory: форматы, группы, group_vars/host_vars, динамический инвентарь |
| [[ansible/explanation/playbook-anatomy]] | Play, Tasks, Handlers, порядок выполнения, import vs include |
| [[ansible/explanation/variables-and-facts]] | Типы переменных, Facts, Register, приоритет (22 уровня) |
| [[ansible/explanation/roles-and-collections]] | Roles: структура, defaults vs vars. Collections: Galaxy, FQCN |
| [[ansible/explanation/async-and-delegation]] | async/poll, fire-and-forget, delegate_to, run_once, wait_for |
| [[ansible/explanation/error-handling]] | block/rescue/always, ignore_errors, failed_when, changed_when, retries |

## How-to (Практические руководства)

| Документ | Описание |
|----------|----------|
| [[ansible/how-to/install-and-configure]] | Установка (pipx/apt/brew), ansible.cfg, pipelining |
| [[ansible/how-to/manage-inventory]] | Создание inventory, SSH-ключи, параметры подключения |
| [[ansible/how-to/write-playbooks]] | Запуск, ad-hoc, loops, when, handlers, tags |
| [[ansible/how-to/use-templates]] | Jinja2 шаблоны: синтаксис, условия, циклы, практические примеры |
| [[ansible/how-to/create-roles]] | Создание ролей, defaults, meta/dependencies, Galaxy |
| [[ansible/how-to/manage-secrets]] | Ansible Vault: шифрование, vault.yml, encrypt_string, CI/CD |
| [[ansible/how-to/debug-playbooks]] | debug, -vvv, --diff, debugger, assert, ansible-lint |
| [[ansible/how-to/deploy-strategies]] | Rolling, Blue-Green, Canary. serial, healthcheck, rollback |
| [[ansible/how-to/test-with-molecule]] | Molecule: тестирование ролей, идемпотентность, CI/CD |

## Recipes (Готовые решения)(В процессе)

 | Рецепт | Описание |
|--------|----------|
| [[ansible/how-to/recipes/server-bootstrap]] | Первичная настройка Ubuntu: users, SSH, swap, UFW |
| [[ansible/how-to/recipes/docker-install]] | Docker CE: GPG, repo, daemon.json hardening |
| [[ansible/how-to/recipes/nginx-certbot]] | Nginx reverse proxy + Let's Encrypt SSL |

## Reference (Справочники)

| Документ | Описание |
|----------|----------|
| [[ansible/reference/modules]] | Справочник модулей: apt, file, template, service, uri, git... |
| [[ansible/reference/jinja2-filters]] | Фильтры Jinja2: default, to_yaml, join, hash, regex, lookups |
| [[ansible/reference/project-structure]] | Рекомендуемая структура проекта, именование, Makefile |
| [[ansible/reference/best-practices]] | Чек-лист: идемпотентность, FQCN, no_log, lint, performance |
| [[ansible/reference/cheatsheet]] | CLI шпаргалка: ansible-playbook, vault, galaxy, inventory, doc |

## Tutorials (Пошаговые уроки)

| # | Документ | Что изучаем |
|---|----------|-------------|
| 01 | [[ansible/tutorials/01-first-playbook]] | От нуля до рабочего playbook (Nginx + handler) |
| 02 | [[ansible/tutorials/02-server-setup]] | Production настройка сервера (SSH, UFW, Docker, Vault) |
| 03 | [[ansible/tutorials/03-roles-project]] | Рефакторинг монолита в 3 роли |
| 04 | [[ansible/tutorials/04-deploy-docker-app]] | Деплой Docker Compose с rollback |

## Быстрый старт

```bash
# 1. Установка
pipx install ansible

# 2. Проверка подключения
ansible all -i "server," -m ping -u ubuntu

# 3. Первый playbook
ansible-playbook site.yml

# 4. С vault
ansible-playbook site.yml --ask-vault-pass
```

## Связанные разделы

- [[docker/index]] — Docker и Docker Compose
- [[kubernetes/index]] — Kubernetes (Phase 3)