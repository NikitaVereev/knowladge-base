---
title: 3 Спецификация Compose (Cheat Sheet)
description: Справочник по основным полям файла compose.yaml (версия Specification). Services, Volumes, Networks и Configs.
---

## Структура верхнего уровня

```yaml
version: "3.8"  # (Устарело в Spec, но часто встречается)
name: my-project # Явное имя проекта

services: ...   # Контейнеры
networks: ...   # Сетевые настройки
volumes: ...    # Именованные тома
secrets: ...    # Секреты (Swarm / Inject)
configs: ...    # Конфигурации (Swarm / Inject)
```

## Service Configuration

Основные поля для блока `services -> my-service`.

### Сборка и Образ
| Поле | Описание |
| :--- | :--- |
| `image` | Имя образа (будет скачан или использован после build). |
| `build` | Путь к папке (`.`) или объект. |
| `build.context` | Путь к контексту сборки. |
| `build.dockerfile` | Имя Dockerfile (если не стандартное). |
| `build.args` | Переменные сборки (`ARG`). |
| `build.target` | Целевой этап (для multi-stage). |

### Запуск и Ресурсы
| Поле | Описание |
| :--- | :--- |
| `command` | Переопределяет `CMD`. |
| `entrypoint` | Переопределяет `ENTRYPOINT`. |
| `container_name` | Жесткое имя контейнера (ломает масштабирование!). |
| `restart` | Политика перезапуска (`unless-stopped`, `on-failure`). |
| `deploy.replicas` | Количество копий (при `--compatibility` или в Swarm). |
| `deploy.resources` | Лимиты (`limits: { cpus: '0.5', memory: '512M' }`). |

### Сеть и Порты
| Поле | Описание |
| :--- | :--- |
| `ports` | Публикация портов (`- "8080:80"`). |
| `expose` | Открыть порт только внутри сети Docker (не наружу). |
| `networks` | Список сетей, к которым подключен сервис. |
| `extra_hosts` | Доп. записи в `/etc/hosts` (`host-gateway`). |
| `dns` | Кастомный DNS сервер. |

### Данные и Конфигурация
| Поле | Описание |
| :--- | :--- |
| `volumes` | Монтирование (`- ./data:/app/data`). |
| `environment` | Переменные (`- DEBUG=1` или словарь). |
| `env_file` | Путь к файлу с переменными (`.env`). |
| `secrets` | Подключение секретов в `/run/secrets`. |
| `healthcheck` | Проверка здоровья (`test: ["CMD", "curl", ...]`). |
| `depends_on` | Порядок запуска (`condition: service_healthy`). |
| `profiles` | Теги для условного запуска (`["dev", "test"]`). |

## Networks

```yaml
networks:
  app-net:
    driver: bridge # Стандартный драйвер
    internal: true # Закрыть доступ в интернет
    ipam: ...      # Настройка подсетей (subnet)
  shared-net:
    external: true # Сеть создана вручную (не удалять при down)
```

## Volumes

```yaml
volumes:
  db-data:
    driver: local
  nfs-data:
    driver_opts: ... # Подключение NFS/CIFS
  backup-vol:
    external: true # Внешний том
```
