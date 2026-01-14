---
title: 10 Docker Swarm Advanced
---

Продвинутые возможности Docker Swarm: networking, secrets, stateful сервисы, healthchecks и отказоустойчивость.

---

## 📚 Содержание

#### **[[01-swarm-deep-dive|01 Архитектура Swarm (Deep Dive)]]**

**Компоненты:**
- Manager ноды (Raft consensus)
- Worker ноды (задачи выполнения)
- Распределение нагрузки
- Service discovery

**Manager State:**
- Raft база данных
- Состояние сервисов
- Балансировка нагрузки
- Переизбрание лидера

---

#### **[[02-swarm-networking|02 Overlay Networking и Service Discovery]]**

**Overlay Network:**
- Что это и как работает
- Изоляция трафика
- Распределённые сервисы
- DNS service discovery

**Практические примеры:**
- Создание overlay сети
- Подключение сервисов
- Внутренняя коммуникация
- Балансировка между репликами

---

#### **[[03-secrets-configs|03 Secrets и Configs]]**

**Управление секретами:**
- Хранение паролей и токенов
- Шифрование at rest
- Распределение на ноды
- Ротация секретов

**Configs (конфигурационные файлы):**
- Хранение конфигов
- Обновление без перезагрузки
- Версионирование

**Примеры:**
- Database пароли
- API ключи
- TLS сертификаты
- Конфиги приложений

---

#### **[[04-stateful-services|04 Stateful Services]]**

**Как работают stateful сервисы в Swarm:**
- Global сервисы (на каждой ноде)
- Replicated with constraints
- Volumes и persistent storage
- State синхронизация

**Примеры:**
- Databases (PostgreSQL, MongoDB)
- Cache (Redis, Memcached)
- Message queues
- Logging aggregators

---

#### **[[05-healthchecks|05 Health Checks и Self-Healing]]**

**Health checks:**
- HEALTHCHECK инструкция в Dockerfile
- Проверка статуса контейнера
- Автоматический restart
- Docker service health

**Restart policies:**
- any (перезагружать всегда)
- failure (при ошибке)
- none (не перезагружать)

**Self-healing:**
- Автоматическое восстановление
- Переміщение контейнеров
- Распределение нагрузки

---

#### **[[06-docker-stack|06 Docker Stack Deployment]]**

**Docker Stack:**
- Развёртывание compose файлов в Swarm
- Сервисы, сети, volumes
- Управление версиями
- Updates и rollbacks

**Полные примеры:**
- Multi-tier приложение
- Database с volumes
- Reverse proxy (nginx)
- Message queue (RabbitMQ)

---

#### **[[07-fault-tolerance|07 Fault Tolerance и Мониторинг]]**

**Отказоустойчивость:**
- Manager node redundancy
- Quorum в Raft
- Worker node failures
- Network partitions

**Мониторинг:**
- docker node ls
- docker service ps
- Resource constraints
- Logging and debugging

---

## 🔗 Структура раздела

```
10-Docker Swarm Advanced (этот файл)
├── 01 Архитектура Swarm (Deep Dive)
├── 02 Overlay Networking
├── 03 Secrets и Configs
├── 04 Stateful Services
├── 05 Health Checks
├── 06 Docker Stack Deployment
└── 07 Fault Tolerance и Мониторинг
```

---

## 🎯 Ключевые Концепции

**Swarm as Orchestrator:**
```
┌─────────────────────────────────────┐
│   Docker Swarm Orchestrator         │
├─────────────────────────────────────┤
│ • Service scheduling                │
│ • Load balancing                    │
│ • Self-healing                      │
│ • Networking                        │
│ • Storage management                │
│ • Security (secrets)                │
└─────────────────────────────────────┘
```

**High Availability:**
```
3+ Manager Nodes
    ├─ Quorum (N/2 + 1)
    ├─ Raft consensus
    └─ State replication
```

---

## 💾 Требования

**На каждой ноде:**
- Docker 20.10+
- Linux kernel 3.10+
- Сетевая изоляция (overlay)

**На Manager нодах:**
- Стабильная сеть
- Low latency между managers
- Минимум 3 для HA

**На Worker нодах:**
- Достаточно памяти для контейнеров
- Стабильное соединение к managers

---

## ✅ Checklist

**После изучения раздела:**
- ✅ Понимаешь Swarm архитектуру (managers, workers, Raft)
- ✅ Создаёшь overlay networks
- ✅ Используешь secrets и configs
- ✅ Развёртываешь stateful сервисы
- ✅ Настраиваешь health checks
- ✅ Используешь docker stack
- ✅ Обрабатываешь failures
- ✅ Готов к production deployment

---

## 🚀 Production Checklist

**Перед production:**
- ✅ 3+ manager nodes
- ✅ Health checks на всех сервисах
- ✅ Backup manager databases
- ✅ Мониторинг (Prometheus/ELK)
- ✅ Secrets в Vault/HashiCorp
- ✅ Logging aggregation
- ✅ Disaster recovery план
- ✅ Регулярные обновления

---
