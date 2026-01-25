---
title: 8 Публикация в Registry
description: Безопасный логин (PAT), правильное тегирование (Git SHA), работа с GitHub Container Registry (GHCR) и перенос образов между реестрами.
---

## 1. Авторизация (Безопасный Login)

Никогда не используйте пароль от аккаунта Docker Hub или GitHub. Используйте **Personal Access Tokens (PAT)**.

### Docker Hub
1.  Создайте токен в настройках аккаунта (права Read/Write).
2.  Войдите, передавая токен через stdin (чтобы он не сохранился в `history` bash):
    ```bash
    echo "dckr_pat_TOKEN" | docker login -u myuser --password-stdin
    ```

### GitHub Container Registry (GHCR)
Для GHCR логин отличается сервером (`ghcr.io`).
```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

## 2. Тегирование и Пуш

### Паттерн "Double Tagging"
Для продакшена хорошей практикой считается пушить один и тот же образ под двумя тегами:
1.  **Уникальный** (Git Commit SHA или SemVer `v1.2.3`) — для неизменяемости.
2.  **Скользящий** (`v1` или `latest`) — для удобства разработчиков.

```bash
# 1. Сборка
docker build -t myapp .

# 2. Создание алиасов (тегов)
docker tag myapp:latest ghcr.io/user/myapp:a1b2c3d  # Git SHA
docker tag myapp:latest ghcr.io/user/myapp:v1       # Major version
docker tag myapp:latest ghcr.io/user/myapp:latest   # Floating tag

# 3. Пуш всех тегов
docker push ghcr.io/user/myapp:a1b2c3d
docker push ghcr.io/user/myapp:v1
docker push ghcr.io/user/myapp:latest
```
*Docker умен: слои (blobs) загрузятся один раз. Теги — это просто легкие ссылки на один манифест.*

## 3. Работа с Приватными Реестрами

Если вы используете Self-Hosted реестр (например, `registry.corp.lan:5000`):

1.  Образ должен начинаться с домена реестра:
    ```bash
    docker tag myapp registry.corp.lan:5000/team/myapp:v1
    docker push registry.corp.lan:5000/team/myapp:v1
    ```

2.  **Проблема Insecure Registry**: Если у вашего реестра нет HTTPS (самоподписанный сертификат), Docker откажется пушить.
    *   *Решение*: Добавить в `/etc/docker/daemon.json`:
        ```json
        {
          "insecure-registries" : ["registry.corp.lan:5000"]
        }
        ```
    *   Перезапустить докер: `sudo systemctl restart docker`.

## 4. Ре-таггинг (Копирование между реестрами)
Сценарий: Забрать образ из публичного Docker Hub и положить в корпоративный Artifactory.

```bash
# 1. Скачать (Pull)
docker pull postgres:15

# 2. Переименовать (Tag)
docker tag postgres:15 private-repo.com/mirror/postgres:15

# 3. Загрузить (Push)
docker push private-repo.com/mirror/postgres:15
```

## 5. Очистка локального кеша
После пуша образы остаются на диске. В CI/CD пайплайнах это может забить место.
```bash
# Удалить локальную ссылку (в реестре останется)
docker rmi ghcr.io/user/myapp:v1
```
