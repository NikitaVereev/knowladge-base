---
title: 03 Secrets и Configs
---

---

## Secrets (Секреты)

```bash
# Создать секрет
echo "my-secret-password" | docker secret create db_password -

# Список секретов
docker secret ls

# Удалить (только если не используется)
docker secret rm db_password

# Использовать в сервисе
docker service create \
  --secret db_password \
  --name app \
  myapp

# В контейнере доступен как файл
# /run/secrets/db_password (только read!)
```

---

## Configs (Конфигурационные файлы)

```bash
# Создать config из файла
docker config create nginx.conf ./nginx.conf

# Список конфигов
docker config ls

# Использовать в сервисе
docker service create \
  --config source=nginx.conf,target=/etc/nginx/nginx.conf \
  --name web \
  nginx

# Обновить конфиг
docker config create nginx.conf.v2 ./nginx.conf.v2
docker service update \
  --config-rm nginx.conf \
  --config-add source=nginx.conf.v2 \
  web
```

---

## Разница между Secrets и Configs

| Aspect | Secrets | Configs |
|--------|---------|---------|
| Шифрование | Encrypted at rest | Plaintext |
| Размер | <500KB | <64KB |
| Обновление | Requires update | Can rotate |
| Использование | Passwords, tokens | Configuration files |
| Версионирование | Нет, удаляй старые | Да, есть версии |

---

## Практический Пример

**docker-compose.yml с secrets:**
```yaml
services:
  web:
    image: myapp
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    external: true
```

**Развернуть:**
```bash
# Создать секрет
echo "secure-password" | docker secret create db_password -

# Deploy stack
docker stack deploy -c docker-compose.yml myapp

# В контейнере:
cat /run/secrets/db_password
# secure-password
```

---

## Ротация Секретов (Best Practice)

```bash
# 1. Создать новый секрет
echo "new-password" | docker secret create db_password_v2 -

# 2. Update service на новый секрет
docker service update \
  --secret-rm db_password \
  --secret-add db_password_v2 \
  app

# 3. Контейнеры перезагружаются с новым секретом

# 4. Удалить старый секрет
docker secret rm db_password
```

---

## Security Best Practices

✓ **Secrets:**
- Никогда не хардкодируй пароли
- Используй secrets в production
- Ротируй регулярно
- Audit access

✓ **Configs:**
- Для публичных конфигов (nginx.conf, app.conf)
- Версионируй конфиги
- Обновляй без restart если возможно

✓ **General:**
- Minimal permissions
- Least privilege
- Backup secrets
- Audit logging

---

**Следующее:** [[04-stateful-services|Stateful Services]]
