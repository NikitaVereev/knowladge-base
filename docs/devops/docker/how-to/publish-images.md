---
title: "Публикация образов"
description: "Как загрузить образ в Docker Hub, GitHub Container Registry или настроить локальный реестр."
---


## Публикация в Docker Hub

### 1. Авторизация
```bash
docker login
# Введите имя пользователя и пароль (или токен)
```

Для CI/CD безопаснее использовать:
```bash
echo "$DOCKER_TOKEN" | docker login -u myusername --password-stdin
```

### 2. Тегирование образа
```bash
# Локальный образ нужно пометить именем, начинающимся с вашего username
docker tag myapp:1.0 myusername/myapp:1.0
```

### 3. Загрузка
```bash
docker push myusername/myapp:1.0
```

## Публикация в GitHub Container Registry

### 1. Создание токена доступа
GitHub → Settings → Developer settings → Personal access tokens (classic)  
Выбрать права: `read:packages`, `write:packages`

### 2. Вход
```bash
echo "ghp_YOUR_TOKEN" | docker login ghcr.io -u your-username --password-stdin
```

### 3. Публикация
```bash
docker tag myapp:1.0 ghcr.io/your-username/myapp:1.0
docker push ghcr.io/your-username/myapp:1.0
```

## Настройка локального реестра

Для разработки или внутренних нужд можно запустить собственный registry.

```bash
docker run -d -p 5000:5000 --name my-registry registry:2
```

Использование:
```bash
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# На другой машине в той же сети:
docker pull 192.168.1.100:5000/myapp:1.0
```

### С персистентным хранилищем (Docker Compose)

```yaml
version: '3.9'
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry

volumes:
  registry-data:
```

```bash
docker-compose up -d
```

## Проверка содержимого реестра

Список образов в локальном реестре:
```bash
curl http://localhost:5000/v2/_catalog
```

Теги конкретного образа:
```bash
curl http://localhost:5000/v2/myapp/tags/list
```

## Связанные материалы

- [[devops/docker/explanation/image-registries|Типы реестров]]
- [[devops/docker/tutorials/ci-cd-basics|Автоматизация сборки и публикации]]
