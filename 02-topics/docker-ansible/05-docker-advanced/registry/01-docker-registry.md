### Docker Registry: Публикация и распространение образов

Распространение Docker образов через различные реестры.

---

#### Что такое Docker Registry

**Docker Registry:**
- Приложение, предоставляющее API для хранения, публикации и извлечения Docker образов
- Централизованное хранилище для образов
- Позволяет обмениваться образами между разработчиками и системами
- Основа для CI/CD pipelines

**Как работает:**
1. При `docker pull` — система запрашивает образ с указанным тегом из registry
2. При `docker push` — образ загружается в registry
3. При `docker run` — если образ не найден локально, он автоматически скачивается

---

#### Docker Hub (официальный реестр)

**Что это:**
- Официальный public registry Docker
- Содержит миллионы готовых образов
- Бесплатно для public образов
- Платные планы для private repositories

**Типы аккаунтов:**
- Personal — для личного использования
- Organization — для команд и компаний
- Teams — управление доступом

**Лимиты на бесплатном плане:**
- Unlimited public repositories
- 1 private repository
- Rate limits: 100 pull/6 часов (anonymous), 200 pull/6 часов (authenticated)

**Платные планы:**
- Pro: $5/месяц — unlimited private repositories
- Team: $9/месяц (per user) — для команд
- Business: custom pricing — для организаций

---

#### Работа с Docker Hub

##### docker login: Аутентификация

**Базовый вход:**
```bash
docker login                                   # интерактивный вход
# Вводишь username и password/token
```

**Вход с параметрами:**
```bash
docker login -u username -p password_or_token
```

**Вход через stdin (для CI/CD):**
```bash
echo "token" | docker login -u username --password-stdin
```

**Выход:**
```bash
docker logout
```

**Проверка статуса:**
```bash
cat ~/.docker/config.json                      # конфиг с учётными данными (зашифрован)
```

---

##### docker tag: Тегирование образов

**Синтаксис:**
```bash
docker tag source_image:tag target_image:tag
```

**Пример:**
```bash
# Собрал образ
docker build -t myapp:1.0 .

# Теггирую для Docker Hub (username/repo:tag)
docker tag myapp:1.0 myusername/myapp:1.0
docker tag myapp:1.0 myusername/myapp:latest   # latest tag для production
```

**Правила тегирования:**

| Registry | Формат | Пример |
|----------|--------|--------|
| **Docker Hub** | `username/repo:tag` | `john/webapp:1.0` |
| **GitHub** | `ghcr.io/username/repo:tag` | `ghcr.io/john/webapp:1.0` |
| **GitLab** | `registry.gitlab.com/user/repo:tag` | `registry.gitlab.com/john/webapp:1.0` |
| **Private** | `registry.example.com/repo:tag` | `registry.example.com/myapp:1.0` |

---

##### docker push: Загрузка образа

**Синтаксис:**
```bash
docker push image_name:tag
```

**Пример:**
```bash
# Push на Docker Hub (нужен login)
docker push myusername/myapp:1.0

# Push на GitHub Container Registry
docker push ghcr.io/myusername/myapp:1.0

# Push на private registry
docker push registry.example.com/myapp:1.0
```

**Важно:**
- Перед push нужен `docker login`
- Образ должен быть правильно теггирован (с username/organization)
- Push может занять время в зависимости от размера образа

**Примеры:**
```bash
# Собраль → теггировать → push
docker build -t myapp:1.0.0 .
docker tag myapp:1.0.0 myusername/myapp:1.0.0
docker tag myapp:1.0.0 myusername/myapp:latest
docker push myusername/myapp:1.0.0
docker push myusername/myapp:latest
```

---

##### docker pull: Скачивание образа

**Синтаксис:**
```bash
docker pull image_name:tag
```

**Примеры:**
```bash
docker pull nginx                              # официальный образ
docker pull myusername/myapp:1.0              # собственный образ
docker pull ghcr.io/username/app:latest       # GitHub registry
docker pull registry.example.com/app:1.0      # private registry
```

**Автоматический pull:**
```bash
docker run -d myusername/myapp:1.0            # pull автоматический если образа нет
```

---

##### docker search: Поиск образов

**Синтаксис:**
```bash
docker search query
```

**Примеры:**
```bash
docker search nginx                            # поиск nginx образов
docker search --limit 5 node                   # ограничить результаты
docker search --filter stars=100 mysql         # фильтр по звёздам
```

**Важно:**
- Поиск работает только для Docker Hub
- Результаты включают: NAME, DESCRIPTION, STARS, OFFICIAL, AUTOMATED

---

#### GitHub Container Registry

**Преимущества:**
- Интегрирован с GitHub
- Private образы можно хранить в приватных репозиториях
- Бесплатно для public образов
- Бесплатно для private образов (до лимита)

**Аутентификация с GitHub:**

```bash
# Создать Personal Access Token (Settings → Developer → Personal access tokens)
# Выбрать: read:packages, write:packages, delete:packages

# Вход через token
echo "your_token" | docker login ghcr.io -u your_username --password-stdin
```

**Push на GitHub:**
```bash
# Теггировать образ
docker tag myapp:1.0 ghcr.io/myusername/myapp:1.0

# Push
docker push ghcr.io/myusername/myapp:1.0

# Pull
docker pull ghcr.io/myusername/myapp:1.0
```

**Пример работы:**
```bash
# 1. Собрать образ
docker build -t myapp:1.0 .

# 2. Теггировать для GitHub
docker tag myapp:1.0 ghcr.io/myusername/myapp:1.0

# 3. Аутентифицироваться
echo "token" | docker login ghcr.io -u myusername --password-stdin

# 4. Push
docker push ghcr.io/myusername/myapp:1.0

# 5. Проверить на GitHub (Settings → Packages and registries)
```

---

#### Локальный Docker Registry

**Когда использовать:**
- Private registry для компании
- Local development и testing
- Offline окружение
- Custom registry с дополнительными функциями

**Запуск локального registry (простой способ):**
```bash
docker run -d -p 5000:5000 --name registry registry:latest
```

**Registry будет доступен на:** `localhost:5000`

**Работа с локальным registry:**
```bash
# Теггировать образ
docker tag myapp:1.0 localhost:5000/myapp:1.0

# Push
docker push localhost:5000/myapp:1.0

# Pull
docker pull localhost:5000/myapp:1.0
```

---

#### Docker Registry с Docker Compose

**Локальный registry с volume (продвинутый):**

```yaml
version: '3.9'

services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry_data:/var/lib/registry
    environment:
      REGISTRY_HTTP_ADDR: 0.0.0.0:5000
    restart: always

volumes:
  registry_data:
```

**Запуск:**
```bash
docker-compose up -d
```

**Работа:**
```bash
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
docker pull localhost:5000/myapp:1.0
```

---

#### Версионирование и Тегирование

**Semantic Versioning (семантическое версионирование):**

Формат: `MAJOR.MINOR.PATCH-PRERELEASE+BUILD`

| Версия | Значение |
|--------|----------|
| `1.0.0` | Первый release |
| `1.0.1` | Patch fix (bug fix) |
| `1.1.0` | Minor release (новые features) |
| `2.0.0` | Major release (breaking changes) |
| `1.0.0-alpha` | Alpha версия |
| `1.0.0-beta.1` | Beta версия |
| `1.0.0-rc.1` | Release candidate |

**Практические примеры:**

```bash
# Собрать и теггировать версию
docker build -t myapp:1.0.0 .

# Также теггировать как latest
docker tag myapp:1.0.0 myusername/myapp:1.0.0
docker tag myapp:1.0.0 myusername/myapp:1.0      # minor version tag
docker tag myapp:1.0.0 myusername/myapp:latest    # latest tag

# Push все версии
docker push myusername/myapp:1.0.0
docker push myusername/myapp:1.0
docker push myusername/myapp:latest
```

**Стратегия версионирования:**
```bash
# Для production
docker build -t myapp:1.0.0 .
docker tag myapp:1.0.0 myusername/myapp:1.0.0
docker tag myapp:1.0.0 myusername/myapp:1.0
docker tag myapp:1.0.0 myusername/myapp:latest
docker push myusername/myapp:1.0.0
docker push myusername/myapp:1.0
docker push myusername/myapp:latest

# Для development (не latest)
docker build -t myapp:dev .
docker tag myapp:dev myusername/myapp:dev
docker push myusername/myapp:dev

# Для testing (beta)
docker build -t myapp:beta .
docker tag myapp:beta myusername/myapp:beta
docker push myusername/myapp:beta
```

---

#### Практические примеры

##### Пример 1: Публикация на Docker Hub

```bash
# 1. Создать Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
EOF

# 2. Собрать образ
docker build -t myapp:1.0.0 .

# 3. Теггировать
docker tag myapp:1.0.0 myusername/myapp:1.0.0
docker tag myapp:1.0.0 myusername/myapp:latest

# 4. Логин на Docker Hub
docker login

# 5. Push образы
docker push myusername/myapp:1.0.0
docker push myusername/myapp:latest

# 6. Проверить на https://hub.docker.com/r/myusername/myapp
```

##### Пример 2: GitHub Container Registry

```bash
# 1. Создать Personal Access Token на GitHub
# Settings → Developer settings → Personal access tokens (classic)
# Выбрать: read:packages, write:packages, delete:packages

# 2. Логин
echo "ghp_xxx..." | docker login ghcr.io -u username --password-stdin

# 3. Теггировать
docker tag myapp:1.0.0 ghcr.io/myusername/myapp:1.0.0

# 4. Push
docker push ghcr.io/myusername/myapp:1.0.0

# 5. Проверить видимость (Settings → Packages and registries)
```

##### Пример 3: Локальный Registry

```bash
# 1. Запустить registry
docker run -d -p 5000:5000 --name my-registry registry:2

# 2. Теггировать образ
docker tag myapp:1.0.0 localhost:5000/myapp:1.0.0

# 3. Push
docker push localhost:5000/myapp:1.0.0

# 4. Проверить
curl http://localhost:5000/v2/_catalog
# {"repositories":["myapp"]}

# 5. Pull образ на другой машине (если в локальной сети)
docker pull 192.168.1.100:5000/myapp:1.0.0
```

##### Пример 4: CI/CD автоматический push

**GitHub Actions:**
```yaml
name: Build and Push

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build image
        run: docker build -t myapp:${{ github.ref_name }} .
      
      - name: Tag image
        run: |
          docker tag myapp:${{ github.ref_name }} ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.ref_name }}
          docker tag myapp:${{ github.ref_name }} ${{ secrets.DOCKER_USERNAME }}/myapp:latest
      
      - name: Login to Docker Hub
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
      
      - name: Push image
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.ref_name }}
          docker push ${{ secrets.DOCKER_USERNAME }}/myapp:latest
```

---

#### Best Practices

✅ **Именование:**
- Используй meaningful имена (не `app`, а `my-service-api`)
- Используй namespace для организации (`myorg/myapp`)

✅ **Версионирование:**
- Используй semantic versioning (MAJOR.MINOR.PATCH)
- Избегай частого обновления `latest` тега
- Помечай важные версии (stable, lts, など)

✅ **Security:**
- Не публикуй sensitive данные в образах
- Используй `.dockerignore` для исключения файлов
- Используй non-root пользователя в контейнерах
- Сканируй образы на уязвимости

✅ **Cleanup:**
- Удаляй старые неиспользуемые образы
- Удаляй dangling образы (`docker image prune`)
- Ограничивай размер реестра

✅ **Storage:**
- Используй external volumes для registry
- Делай regular backups
- Мониторь размер реестра

✅ **Production:**
- Используй TLS/SSL для registry
- Включи аутентификацию
- Используй private repository для чувствительных образов
- Документируй образы в README
