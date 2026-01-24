---
title: "8 Публикация образов"
description: "Как отправить (push) образ в Docker Hub и GitHub Container Registry (GHCR). Настройка токенов доступа (PAT)."
---

После того как образ собран локально, его нужно опубликовать в реестре, чтобы он стал доступен на сервере или для коллег.

## 1. Docker Hub

### Шаг 1: Создание токена
1.  Зайдите на [hub.docker.com](https://hub.docker.com).
2.  Настройки аккаунта -> **Security** -> **New Access Token**.
3.  Дайте имя (например, "MacBook CLI") и права (Read/Write).
4.  Скопируйте токен (он начинается на `dckr_pat_...`).

### Шаг 2: Логин
```bash
docker login -u ваш_юзернейм
# Вставьте токен вместо пароля
```
*Если успешно: `Login Succeeded`.*

### Шаг 3: Тегирование и Push
Docker понимает, куда отправлять образ, только по его имени. Имя должно начинаться с вашего логина.

```bash
# 1. Вешаем "биркy" (tag) с правильным именем
docker tag my-app:latest username/my-app:v1

# 2. Отправляем
docker push username/my-app:v1
```

## 2. GitHub Container Registry (GHCR)

Этот реестр удобнее, если ваш код уже лежит на GitHub. Образы будут привязаны к репозиторию.

### Шаг 1: Токен (GitHub PAT)
1.  GitHub -> Settings -> Developer Settings -> **Personal access tokens (Classic)**.
2.  Создайте токен с правами:
    *   `write:packages` (для загрузки образов)
    *   `read:packages` (для скачивания)
    *   `delete:packages` (опционально)
3.  Скопируйте токен (он начинается на `ghp_...`).

### Шаг 2: Логин
```bash
# Лучше передавать пароль через stdin, чтобы он не остался в истории bash
echo "ghp_ВАШ_ТОКЕН" | docker login ghcr.io -u ВАШ_GITHUB_USER --password-stdin
```

### Шаг 3: Push
Адрес GHCR всегда включает домен `ghcr.io`.

```bash
# Тегируем
docker tag my-app:latest ghcr.io/username/repo-name:v1

# Отправляем
docker push ghcr.io/username/repo-name:v1
```

> **Совет для GitHub Actions:** Внутри CI/CD пайплайнов создавать токен руками не нужно. GitHub автоматически предоставляет секрет `${{ secrets.GITHUB_TOKEN }}`, который уже имеет права на запись в реестр текущего репозитория.
