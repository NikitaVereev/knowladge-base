---
title: 8 Работа с Private Registry
type: how-to
tags: [docker, registry, ghcr, authentication, secrets, config-json]
---

# Работа с приватными реестрами (Private Registry)

В корпоративной среде образы хранятся в приватных реестрах (GitHub Container Registry, GitLab Registry, Nexus, Harbor, AWS ECR). Docker требует аутентификации для доступа к ним.

## 1. Логин (docker login)

Никогда не используйте пароль от аккаунта! Используйте **Access Tokens**.

### GitHub Container Registry (GHCR)
1.  Перейдите в GitHub -> Settings -> Developer Settings -> Personal access tokens (Classic).
2.  Создайте токен с правами `read:packages` (для скачивания) и `write:packages` (для загрузки).
3.  Сохраните токен в файл или переменную.

```bash
# Логин через STDIN (безопасно, не остается в bash history)
echo $GH_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin
```
*Успех: `Login Succeeded`. Конфиг сохранится в `~/.docker/config.json`.*

### AWS ECR
AWS использует временные токены (живут 12 часов).
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

## 2. Использование в Docker Compose

Docker Compose автоматически использует креды из `~/.docker/config.json` хоста. Никакой дополнительной настройки не нужно.

```yaml
services:
  app:
    # Просто укажите полный путь к образу
    image: ghcr.io/my-org/my-private-app:v1.0
```

## 3. Pull Secrets на сервере (CI/CD)

Как дать серверу доступ к реестру, не логинясь вручную?

### Вариант А: Config JSON (Docker)
Создайте на сервере файл `~/.docker/config.json` с base64-кодированным auth string.

```json
{
	"auths": {
		"ghcr.io": {
			"auth": "BASE64_STRING_HERE" # echo -n "user:token" | base64
		}
	}
}
```

### Вариант Б: CI/CD Variables
В GitLab CI или GitHub Actions используйте встроенные механизмы.
*   **GitHub Actions:** `docker/login-action` (см. гайд по Pipeline).
*   **GitLab CI:** Переменная `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` доступна автоматически.

## 4. Запуск собственного Registry

Иногда нужно поднять свой простой реестр (например, внутри закрытого VPN).

```bash
# 1. Запуск реестра на порту 5000
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# 2. Тегирование образа
docker tag my-app:latest localhost:5000/my-app:v1

# 3. Пуш
docker push localhost:5000/my-app:v1
```

### Проблема HTTPS (Insecure Registry)
По умолчанию Docker требует HTTPS для всех реестров, кроме `localhost`. Если вы хотите пушить в `http://192.168.1.50:5000` (без SSL), нужно разрешить это в настройках демона.

**`/etc/docker/daemon.json`:**
```json
{
  "insecure-registries" : ["192.168.1.50:5000"]
}
```
*Не делайте так в продакшене! Используйте Nginx с Let's Encrypt перед реестром.*

## 5. Работа с Docker Hub (Rate Limits)

У Docker Hub есть жесткие лимиты на скачивание для анонимов (100 пуллов за 6 часов).
Если ваш CI падает с ошибкой `toomanyrequests`, обязательно настройте авторизацию даже для публичных образов.

1.  Зарегистрируйте бота на Docker Hub.
2.  Сделайте `docker login`.
3.  Лимит увеличится до 200 (Free) или станет безлимитным (Pro).
