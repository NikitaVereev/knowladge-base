---
title: "Использование Handlers"
description: "Как работают уведомления (notify), цепочки handlers и группировка через listen."
---


Handlers (обработчики) — это специальные задачи, которые запускаются только при получении уведомления (`notify`) от других задач.

**Ключевое правило:** Handler выполняется **один раз** в конце плейбука, даже если его уведомили 10 раз.

## Базовый пример
Перезагрузка Nginx при изменении конфига.

```yaml
tasks:
  - name: Update Nginx config
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: restart nginx

handlers:
  - name: restart nginx
    service: name=nginx state=restarted
```

## Множественные уведомления
Одна задача может триггерить несколько хендлеров.

```yaml
  - name: Update certs
    copy: src=cert.pem dest=/etc/ssl/
    notify:
      - reload nginx
      - restart haproxy
```

## Группировка через `listen`
Можно привязать несколько хендлеров к одному событию через топик `listen`.

```yaml
tasks:
  - name: Update app config
    template: src=app.j2 dest=/etc/app.conf
    notify: "restart app stack"

handlers:
  - name: restart app
    service: name=app state=restarted
    listen: "restart app stack"

  - name: restart monitoring agent
    service: name=agent state=restarted
    listen: "restart app stack"
```

## Связанные материалы
- [[devops/ansible/explanation/playbook-basics|Структура Playbook]]
