---
title: "3 Структура Ansible-проекта"
description: "Рекомендованная иерархия директорий для продакшн-проектов."
---

```text
ansible-project/
├── ansible.cfg              # Конфиг проекта
├── requirements.yml         # Зависимости (Galaxy)
├── inventory/
│   ├── production.yml       # Инвентарь продакшена
│   └── staging.yml          # Инвентарь стейджинга
├── group_vars/              # Переменные групп
│   ├── all.yml              # Общие для всех
│   ├── webservers.yml       # Для группы webservers
│   └── db.yml
├── roles/                   # Роли
│   ├── common/              # Базовая настройка (users, ssh)
│   ├── nginx/
│   └── app/
├── playbooks/               # Плейбуки
│   ├── site.yml             # Главная точка входа
│   ├── deploy.yml           # Деплой приложения
│   └── maintenance.yml      # Обслуживание
└── templates/               # Глобальные шаблоны
```
