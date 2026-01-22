---
title: "9 Использование Handlers (Обработчиков)"
description: "Как перезагружать сервисы только при изменении конфигурации."
---

Handlers — это задачи, которые "слушают" события.

**Правила работы:**
1.  Handler запускается только если его уведомили (`notify`).
2.  Handler запускается только если задача, которая его уведомила, вернула статус `changed`.
3.  Handler запускается **один раз** в конце плейбука (даже если его уведомили 10 раз).

## Пример

```yaml
tasks:
  - name: Update Nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart Nginx

  - name: Update Site config
    template:
      src: site.conf.j2
      dest: /etc/nginx/conf.d/site.conf
    notify: Restart Nginx

handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```

Если оба шаблона изменились, Nginx перезагрузится один раз. Если ничего не изменилось — не перезагрузится вообще.
