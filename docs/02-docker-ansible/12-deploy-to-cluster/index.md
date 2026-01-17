---
title: 12 Deploy to Cluster
---

Практическое развёртывание приложений на Docker Swarm кластеры с использованием Ansible.

---

## 📚 Содержание

#### **[[01-app-preparation|01 Application Preparation]]**

**Подготовка приложения:**
- Dockerfile оптимизация
- Image versioning и tagging
- Multi-stage builds
- Registry setup
- Image repository management

**Примеры:**
- Node.js приложение
- Python/Django приложение
- Go микросервис

---

#### **[[02-swarm-deployment|02 Docker Swarm Deployment]]**

**Развёртывание на Swarm:**
- Service definition
- Stack deployment
- Replicas и constraints
- Resource limits
- Service discovery

**Практика:**
- Multi-service stack
- Environment variables
- Secrets management
- Service updates

---

#### **[[03-health-monitoring|03 Health Checks & Monitoring]]**

**Мониторинг здоровья:**
- Health checks в Docker
- Service readiness
- Metrics collection
- Status verification
- Troubleshooting

**Инструменты:**
- Docker health checks
- docker service ps
- docker service logs
- Custom checks

---

#### **[[04-multi-environment|04 Multi-Environment Deployment]]**

**Управление окружениями:**
- Development environment
- Staging environment
- Production environment
- Configuration differences
- Secrets per environment

**Workflow:**
- Environment-specific playbooks
- Variable files organization
- Deployment automation

---

#### **[[05-updates-rollback|05 Updates & Rollback Strategies]]**

**Обновления и откаты:**
- Service updates
- Rolling updates
- Blue-green deployment
- Canary deployment
- Automatic rollback
- Health check validation

**Примеры:**
- Zero-downtime updates
- Gradual rollout
- Quick rollback on failure

---

#### **[[06-complete-deployment|06 Complete Deployment Project]]**

**Полный проект:**
- End-to-end deployment
- Multiple services
- Database migrations
- Persistent storage
- Load balancing
- Monitoring stack
- Disaster recovery

**Реальный пример:**
- Web tier (Nginx)
- App tier (Node.js/Python)
- Database (PostgreSQL)
- Cache (Redis)
- Monitoring

---

## 🔗 Структура раздела

```
12-Deploy to Cluster (этот файл)
├── 01 Application Preparation (images, versioning)
├── 02 Swarm Deployment (services, stacks)
├── 03 Health Checks & Monitoring (health, status)
├── 04 Multi-Environment (dev, staging, prod)
├── 05 Updates & Rollback (deployment strategies)
└── 06 Complete Project (end-to-end example)
```

---

## 🎯 Ключевые Навыки

**После этого раздела:**
- ✅ Готовишь приложение к production
- ✅ Развёртываешь сервисы на Swarm
- ✅ Настраиваешь health checks
- ✅ Управляешь multiple окружениями
- ✅ Делаешь безопасные обновления
- ✅ Откатываешь при проблемах
- ✅ Мониторишь кластер
- ✅ Готов к production deployment

---

## 💾 Требования

**Инфраструктура:**
- Docker Swarm кластер (3+ узла)
- Ansible на контроль машине
- Git репозиторий для кода
- Container registry (Docker Hub или private)

**Инструменты:**
- Docker Engine 20.10+
- Ansible 2.15+
- Python 3.8+
- Git

---

## 🚀 Workflow Deployment

```
1. Application Preparation
   └─ Build & Push Image

2. Configure Environment
   └─ Set variables & secrets

3. Health Checks Setup
   └─ Define readiness checks

4. Deploy to Swarm
   └─ Run deployment playbook

5. Validate & Monitor
   └─ Check service status

6. Update Service
   └─ New version deployment

7. Rollback if needed
   └─ Restore previous version
```

---

## ✅ Deployment Checklist

**Pre-deployment:**
- [ ] Image built & tested
- [ ] Image pushed to registry
- [ ] Secrets configured
- [ ] Health checks defined
- [ ] Resource limits set
- [ ] Monitoring setup
- [ ] Backup configured

**Post-deployment:**
- [ ] Services running
- [ ] Health checks passing
- [ ] Logs normal
- [ ] Status verified
- [ ] Rollback tested

---