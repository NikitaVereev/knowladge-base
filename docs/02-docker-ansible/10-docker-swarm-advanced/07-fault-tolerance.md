---
title: 07 Fault Tolerance и Мониторинг
---

---

## Manager Quorum

```
N Managers → Quorum = N/2 + 1

1 manager → можно потерять 0 (хрупко!)
3 managers → можно потерять 1 (рекомендуется)
5 managers → можно потерять 2
7 managers → можно потерять 3

ПРАВИЛО: всегда 3, 5, или 7 managers!
```

---

## Восстановление Manager

```bash
# Если manager вышел из строя
# (диск full, corruption, crash)

# 1. Удалить ноду из swarm
docker node rm node1

# 2. Очистить данные на ноде
rm -rf /var/lib/docker/swarm

# 3. Переинициализировать (как worker или новый manager)
docker swarm init --force-new-cluster  # переделать на manager

# 4. Добавить обратно как worker
docker swarm join-token worker
docker swarm join --token TOKEN IP:2377
```

---

## Мониторинг Узлов

```bash
# Список узлов
docker node ls

# Информация о узле
docker node inspect node1

# Статусы:
# ready - готов к контейнерам
# down - не доступен

# Availability:
# active - активна, принимает контейнеры
# pause - паузирована, запущенные работают
# drain - выводится, контейнеры перемещаются

# Drain перед обслуживанием
docker node update --availability drain node1

# Включить обратно
docker node update --availability active node1
```

---

## Мониторинг Сервисов

```bash
# Список сервисов
docker service ls

# Статус сервиса
docker service ps web

# Логирование ошибок
docker service logs web -f

# События swarm
watch -n 1 'docker service ps web'
```

---

## Network Partition Handling

```
Scenario: 3 managers, network split 2 vs 1

Side 1 (2 managers):
- Quorum есть (нужно 2 из 3)
- Cluster работает ✓
- Worker nodes на этой стороне OK

Side 2 (1 manager):
- Quorum нет (нужно 2)
- Cluster неработоспособен ✗
- Worker nodes тут работают, но не могут обновляться
```

---

## Logging & Debugging

```bash
# Структурированные логи
journalctl -u docker -f

# Логи swarm events
docker events --filter type=service

# Inspect swarm DB
docker system df   # disk usage

# Проверить сеть
docker network inspect <network>

# Проверить volumes
docker volume ls
docker volume inspect <volume>
```

---

## Backup Manager DB

```bash
# Локальный backup
docker run --rm \
  -v /var/lib/docker/swarm:/swarm \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/swarm-backup.tar.gz /swarm

# Восстановление при disaster
docker run --rm \
  -v /var/lib/docker:/var/lib/docker \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/swarm-backup.tar.gz -C /

docker swarm init --force-new-cluster
```

---

## Production Checklist

✅ **High Availability:**
- [ ] 3+ managers (нечётное число)
- [ ] Managers на разных физических хостах
- [ ] Резервные каналы между managers

✅ **Monitoring:**
- [ ] docker node ls регулярно
- [ ] docker service ps регулярно
- [ ] Health checks на сервисах
- [ ] Мониторинг (Prometheus + Grafana)

✅ **Backup:**
- [ ] Ежедневный backup manager DB
- [ ] Хранить off-site
- [ ] Тестировать восстановление

✅ **Updates:**
- [ ] Drain node перед обновлением
- [ ] Обновлять по одному
- [ ] Мониторить логи

✅ **Logging:**
- [ ] Centralized logging (ELK, Splunk)
- [ ] Audit logs
- [ ] Application logs

---

## Когда Использовать Swarm vs Kubernetes

**Swarm (когда):**
- ✓ Простая инфраструктура
- ✓ 5-50 узлов
- ✓ Нет сложных требований
- ✓ Native Docker commands

**Kubernetes (когда):**
- ✓ Масштабирование 100+ узлов
- ✓ Сложные сетевые требования
- ✓ Custom resource types
- ✓ Enterprise support нужен
- ✓ Multi-region deployment

---