---
title: "Ansible Best Practices"
description: "Как правильно структурировать проект, именовать переменные и использовать Git."
---


Чтобы ваш проект не превратился в свалку YAML-файлов, следуйте этим рекомендациям.

## 1. Структура проекта
Используйте стандартную иерархию директорий:

```text
ansible-project/
├── ansible.cfg             # Конфиг проекта
├── hosts.yaml              # Инвентарь
├── playbooks/              # Точки входа (site.yml, deploy.yml)
├── roles/                  # Самодостаточные роли (nginx, db)
├── group_vars/             # Переменные для групп (webservers.yml)
├── host_vars/              # Переменные для хостов (web1.yml)
└── templates/              # Jinja2 шаблоны
```

## 2. Именование переменных
Используйте префиксы, чтобы избежать коллизий имен, так как Ansible имеет плоское пространство имен переменных.

* **Плохо:** `port: 80`, `user: admin` (слишком общие имена, могут быть перезаписаны).
* **Хорошо:** `nginx_port: 80`, `app_db_user: admin`.

## 3. Идемпотентность и Проверки
* Всегда используйте `--check` (Dry Run) перед запуском на проде.
* Используйте `block/rescue` для обработки ошибок и отката изменений.

```yaml
- block:
    - name: Deploy app
      command: /opt/deploy.sh
  rescue:
    - name: Rollback
      command: /opt/rollback.sh
```

## 4. Версионирование
Храните весь код инфраструктуры в Git. Не храните секреты (пароли, ключи) в открытом виде — используйте **Ansible Vault**.

## Связанные материалы
- [[devops/ansible/tutorials/playbook-examples|Примеры готовых Playbooks]]
- [[devops/ansible/explanation/inventory-basics|Организация инвентаря]]
