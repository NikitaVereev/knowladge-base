---
title: "Стратегии версионирования образов"
description: "Semantic Versioning для Docker-образов. Использование тегов latest, stable, dev."
---


## Semantic Versioning (SemVer)

Формат: `MAJOR.MINOR.PATCH`

| Компонент | Когда увеличивается |
|-----------|---------------------|
| MAJOR | Несовместимые изменения API |
| MINOR | Новая функциональность, обратная совместимость |
| PATCH | Исправления ошибок |

Примеры:
- `1.0.0` — первый стабильный релиз
- `1.0.1` — hotfix
- `1.1.0` — новая функция
- `2.0.0` — breaking changes

## Рекомендации по тегированию

### Для production-образа
```bash
docker build -t myapp:1.2.3 .

# Теггировать несколько версий:
docker tag myapp:1.2.3 myusername/myapp:1.2.3   # Точная версия
docker tag myapp:1.2.3 myusername/myapp:1.2     # Minor (обновляется при 1.2.x)
docker tag myapp:1.2.3 myusername/myapp:latest  # Последний стабильный

docker push myusername/myapp:1.2.3
docker push myusername/myapp:1.2
docker push myusername/myapp:latest
```

### Для development-образа
```bash
docker build -t myapp:dev .
docker tag myapp:dev myusername/myapp:dev
docker push myusername/myapp:dev
```

### Pre-release версии
- `1.0.0-alpha` — ранняя альфа-версия
- `1.0.0-beta.1` — бета-тестирование
- `1.0.0-rc.1` — release candidate

## Что делать с тегом `latest`

**Опасность:** Тег `latest` не означает "самая свежая версия", а лишь "образ без явного тега". Он может указывать на устаревшую версию.

**Рекомендация:**
- В продакшене **всегда** указывайте конкретную версию (`myapp:1.2.3`)
- Обновляйте `latest` только после стабилизации релиза
- Для dev-окружений можно использовать отдельный тег (`dev`, `nightly`)

## Связанные материалы

- [[devops/docker/how-to/publish-images|Публикация образов]]
- [[devops/docker/tutorials/dockerfile-basics|Написание Dockerfile]]
