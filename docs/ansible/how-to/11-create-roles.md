---
title: "11 Создание и использование ролей"
description: "Полное руководство по структуре ролей, зависимостям и Ansible Galaxy."
---

Роли — это основной способ организации кода в Ansible. Они позволяют переиспользовать логику и делиться ею.

## Структура роли

Создайте роль командой: `ansible-galaxy init my_role`

```text
my_role/
├── tasks/
│   └── main.yml      # Главный список задач (entry point)
├── handlers/
│   └── main.yml      # Обработчики (сервисы)
├── defaults/
│   └── main.yml      # Переменные по умолчанию (легко переопределить)
├── vars/
│   └── main.yml      # Переменные роли (трудно переопределить)
├── files/            # Статические файлы (для модуля copy)
├── templates/        # Шаблоны Jinja2 (для модуля template)
├── meta/
│   └── main.yml      # Метаданные и зависимости
└── README.md         # Документация
```

## Вызов роли в Playbook

```yaml
- hosts: webservers
  roles:
    # 1. Простой вызов
    - my_role

    # 2. Вызов с передачей параметров
    - role: my_role
      vars:
        port: 8080
        app_state: present
    
    # 3. Условный вызов
    - role: monitoring
      when: enable_monitoring | bool
```

## Управление зависимостями (`meta/main.yml`)
Если ваша роль зависит от другой (например, `php` требует `nginx`), опишите это в метаданных. Ansible автоматически запустит зависимости перед вашей ролью.

```yaml
dependencies:
  - role: geerlingguy.nginx
  - role: geerlingguy.php
    vars:
      php_version: '8.2'
```

## Использование Ansible Galaxy
Galaxy — это хаб готовых ролей.

1.  **Поиск роли:** `ansible-galaxy search nginx`
2.  **Установка:** `ansible-galaxy install geerlingguy.nginx`
3.  **Установка через requirements.yml:**
    Создайте файл `requirements.yml`:
    ```yaml
    roles:
      - name: geerlingguy.redis
        version: "1.8.0"
      - src: https://github.com/myorg/private-role.git
        name: private_role
    ```
    Запустите: `ansible-galaxy install -r requirements.yml`
