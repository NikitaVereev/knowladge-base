---
title: "10 Использование тегов (Tags)"
description: "Как запускать или пропускать определенные части Playbook."
---

Теги позволяют запускать подмножество задач из большого плейбука.

## Назначение тегов

```yaml
tasks:
  - name: Install Packages
    apt: name=nginx
    tags: 
      - install
      - setup

  - name: Configure Nginx
    template: src=nginx.conf dest=/etc/nginx/nginx.conf
    tags: 
      - config
      - nginx
```

## Запуск

1.  **Запустить только конфигурацию:**
    ```bash
    ansible-playbook site.yml --tags "config"
    ```
2.  **Пропустить конфигурацию:**
    ```bash
    ansible-playbook site.yml --skip-tags "config"
    ```
3.  **Посмотреть доступные теги:**
    ```bash
    ansible-playbook site.yml --list-tags
    ```

## Специальные теги
*   `always`: Задача выполнится всегда, если явно не пропущена.
*   `never`: Задача никогда не выполнится, если явно не вызвана (`--tags never`). Удобно для опасных задач типа "удалить базу".
