---
title: 05 Продвинутые возможности
---

Сетевое взаимодействие, хранилище данных и registry.

---

## 📚 Содержание

### **Подраздел: Networking**

#### **[[docs/_inbox/02-docker-ansible/05-docker-advanced/networking/index|01 Docker Networking]]**

**Docker Networking:**
- Основы Docker сетей (LibNetwork)
- Типы сетевых драйверов
- Bridge, Host, Overlay, Macvlan, None
- Управление сетями (create, connect, inspect, rm)

**Bridge сеть (default):**
- Соединение контейнеров на одном хосте
- Service Discovery по имени контейнера
- Порт-маппинг для доступа извне
- Практические примеры

**Advanced сетевые режимы:**
- Host Network (прямой доступ к хосту)
- Null Network (без сети)
- Когда использовать каждый режим

**DNS в Docker:**
- Встроенный DNS сервер
- Service Discovery
- Кастомный DNS для контейнеров
- Проблемы и решения

---

### **Подраздел: Storage**

#### **[[docs/_inbox/02-docker-ansible/05-docker-advanced/storage/index|02 Docker Storage and Volumes]]**

**Docker Volumes:**
- Типы хранилища (bind mounts, volumes, tmpfs)
- Named volumes vs anonymous
- Управление volumes

**Named Volumes:**
- Создание и управление
- Монтирование в контейнеры
- Backup и restore

**Bind Mounts:**
- Прямое монтирование директорий
- read-only и read-write
- Для development и production

**Практические примеры:**
- Сохранение данных БД (PostgreSQL, MongoDB)
- Развитие приложения (development volumes)
- Backup и миграция данных

---

### **Подраздел: Registry**

#### **[[docs/_inbox/02-docker-ansible/05-docker-advanced/registry/index|03 Docker Registry]]**

**Docker Registry:**
- Docker Hub (официальный реестр)
- Push и pull образов
- Private registry
- Аутентификация

**Практические примеры:**
- Создание аккаунта на Docker Hub
- Публикация своих образов
- Использование private registry

---

## 🔗 Структура раздела

```
05-Продвинутые возможности   (этот файл)
├── 01 Docker Networking
│   └── 01 Docker Network - Основы и режимы
├── 02 Docker Storage and Volumes
│   └── 01 Docker Storage - Volumes и Bind Mounts
└── 03 Docker Registry
    └── 01 Docker Registry - Публикация и распространение образов
```

---

## 🔗 Связи

**Предыдущие разделы:**
- [[docs/_inbox/02-docker-ansible/04-docker-images-dockerfile/index|04 Образы и Dockerfile]]

**Следующие разделы:**
- [[docs/_inbox/02-docker-ansible/06-docker-compose/index|06 Docker compose]] 

---