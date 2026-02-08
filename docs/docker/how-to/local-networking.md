---
title: "Локальная сеть (Host & SSH)"
type: how-to
tags: [docker, networking, host, ssh-agent, localhost, development]
sources:
  docs: "https://docs.docker.com/network/drivers/host/"
related:
  - "[[docker/explanation/networking]]"
  - "[[docker/how-to/debug-containers]]"
  - "[[docker/tutorials/04-networking-workshop]]"
---

# Локальная сеть и доступ к хосту

> **TL;DR:** Хост из контейнера: `host.docker.internal`. SSH-ключи в build: `--ssh default`.
> SSH в runtime: mount сокета агента.

При локальной разработке часто возникают две проблемы:
1.  Как из контейнера подключиться к сервису, запущенному на хосте (например, к локальному Postgres или API, запущенному без Docker)?
2.  Как пробросить SSH-ключи в контейнер, чтобы сделать `git clone` из приватного репозитория?

## 1. Доступ к хосту из контейнера (`host.docker.internal`)

Вам нужно постучаться из контейнера на `localhost:5432` вашего ноутбука. Но `localhost` внутри контейнера — это сам контейнер.

### Windows / macOS
Docker Desktop автоматически настраивает DNS-запись `host.docker.internal`, которая указывает на IP-адрес хоста.

```bash
# Внутри контейнера
ping host.docker.internal
# PING host.docker.internal (192.168.65.2) ...
```
*Просто используйте `host.docker.internal` вместо `localhost` или `127.0.0.1` в конфигах вашего приложения.*

### Linux
На Linux эта магия по умолчанию выключена (так как Docker работает нативно).
Чтобы включить её, добавьте флаг `--add-host` при запуске:

**Docker CLI:**
```bash
docker run --add-host host.docker.internal:host-gateway my-app
```

**Docker Compose:**
```yaml
services:
  app:
    image: my-app
    extra_hosts:
      - "host.docker.internal:host-gateway"
```
*Теперь `host.docker.internal` работает и на Linux.*

## 2. Доступ к контейнеру с хоста (Port Mapping)

Это стандартная операция, но есть нюанс безопасности.

**Публичный доступ (Опасно):**
```bash
docker run -p 8080:80 nginx
```
*Сервис доступен на `0.0.0.0:8080` (все интерфейсы).*

**Локальный доступ (Безопасно):**
```bash
docker run -p 127.0.0.1:8080:80 nginx
```
*Сервис доступен ТОЛЬКО с вашей машины. Никто из локальной сети (Wi-Fi) не сможет подключиться.*

## 3. Проброс SSH-агента (SSH Forwarding)

Вы хотите сделать `npm install` из приватного Git-репозитория внутри Docker build.
**Антипаттерн:** Копировать `id_rsa` внутрь образа (`COPY ~/.ssh/id_rsa ...`). **Никогда так не делайте!** Ключ останется в слоях образа.

**Правильное решение:** Использовать BuildKit SSH mount.

### Шаг 1: Dockerfile
Используйте специальный тип mount:

```dockerfile
# Синтаксис: --mount=type=ssh
RUN --mount=type=ssh \
    git clone git@github.com:my-company/private-repo.git
```

### Шаг 2: Сборка
Передайте сокет вашего SSH-агента при сборке:

```bash
# Убедитесь, что агент запущен и ключи добавлены (ssh-add)
eval $(ssh-agent)
ssh-add

# Сборка с пробросом агента
docker build --ssh default .
```

## 4. Проброс SSH-агента в Runtime (для Dev Container)

Если вы разрабатываете внутри запущенного контейнера (Dev Containers) и хотите делать `git push`.

**Docker Compose:**
Вам нужно пробросить сокет SSH-агента как Volume и установить переменную `SSH_AUTH_SOCK`.

```yaml
services:
  dev-env:
    image: ubuntu
    volumes:
      # Linux:
      - $SSH_AUTH_SOCK:/ssh-agent
    environment:
      - SSH_AUTH_SOCK=/ssh-agent
```

*На macOS (Docker Desktop) проброс сокета работает "магически" через настройки: `/run/host-services/ssh-auth.sock`.*

## 5. Временный туннель (Ngrok альтернатива)

Если вам нужно временно показать локальный контейнер коллеге через интернет, не обязательно деплоить его на сервер.

Используйте **Cloudflare Tunnel** (бесплатно):

```bash
docker run --network host cloudflare/cloudflared:latest tunnel --url http://localhost:8080
```
*Вы получите временную ссылку `https://...trycloudflare.com`, которая ведет прямо в ваш локальный контейнер.*

## Типичные ошибки

| Ошибка | Симптом | Решение |
|--------|---------|---------|
| `host.docker.internal` на Linux | Не резолвится (работает только на Docker Desktop) | Добавить `--add-host=host.docker.internal:host-gateway` |
| SSH-ключ скопирован в образ | Ключ виден в `docker history` | Использовать `--mount=type=ssh` (BuildKit) — ключ не попадает в слои |
| SSH agent не запущен | `Could not open a connection to your authentication agent` | `eval $(ssh-agent -s) && ssh-add` перед docker build |
| Порт уже занят | `bind: address already in use` | Проверить `lsof -i :PORT`, остановить конфликтующий процесс |