---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# 05: Мониторинг и логирование

## 🎯 ПРИНЦИПЫ МОНИТОРИНГА

### KISS (Keep It Simple, Stupid)
- Используйте встроенные инструменты
- Не переусложняйте
- Мониторьте важное

---

## 📊 ВСТРОЕННЫЕ ИНСТРУМЕНТЫ systemd

### journalctl: логи всего

```bash
# Последние 50 строк
journalctl -n 50

# Лайв логи (tail -f)
journalctl -f

# Логи сегодня
journalctl --since today

# Логи за час
journalctl --since "1 hour ago"

# Конкретный сервис
journalctl -u docker

# С ошибками
journalctl -u docker --priority=err

# ✅ АКТУАЛЬНО: Очистить старые логи
sudo journalctl --vacuum-time=3d

# ❌ СТАРАЯ КОМАНДА (не работает)
sudo journalctl --vacuum=3d

# Disk usage
journalctl --disk-usage
```

### systemd-analyze: производительность

```bash
# Время загрузки всей системы
systemd-analyze

# Критический путь (самые медленные сервисы)
systemd-analyze critical-chain

# Сервисы по времени загрузки
systemd-analyze blame

# График загрузки (SVG)
systemd-analyze plot > /tmp/boot.svg
```

### systemctl: статус сервисов

```bash
# Список всех сервисов
systemctl list-units --type=service

# Failed сервисы
systemctl list-units --state=failed

# Автозагрузка сервисов
systemctl list-unit-files --state=enabled

# Таймеры
systemctl list-timers

# Зависимости сервиса
systemctl list-dependencies docker
```

---

## 📈 СИСТЕМНЫЕ МЕТРИКИ

### Процессы и память

```bash
# Показать процессы
ps aux | head -20

# Top (интерактивное)
top

# Занятая память
free -h

# Per-process память
ps -eo user,pid,vsz,rss,comm --sort=-rss | head -20
```

### Диск

```bash
# Использование диска
df -h

# Размер директории
du -sh /home/*

# Du с limit (top 10)
du -sh /home/* | sort -rh | head -10

# Inode использование
df -i
```

### Сеть

```bash
# IP адреса
ip addr show

# Маршруты
ip route show

# Netstat (какие порты слушают)
netstat -tulpn

# ss (быстрее чем netstat)
ss -tulpn

# Трафик (интерфейсы)
ip -s link
```

---

## 🔧 sysstat: РАСШИРЕННЫЕ МЕТРИКИ

### Команды

```bash
# iostat (диск I/O)
iostat -x 1 5    # Each second, 5 times

# sar (system activity report)
sar 1 5          # Each second, 5 times

# sar CPU
sar -u 1 5

# sar memory
sar -r 1 5

# sar disk
sar -d 1 5
```

---

## 📝 ПРИМЕРЫ МОНИТОРИНГА

### Проверить систему

```bash
#!/bin/bash
# system-check.sh

echo "=== System Health Check ==="

# Uptime
echo "Uptime: $(uptime -p)"

# Memory
echo "Memory: $(free -h | grep Mem | awk '{print $3 "/" $2}')"

# Disk
echo "Disk: $(df -h / | tail -1 | awk '{print $3 "/" $2}')"

# Failed services
echo "Failed services:"
systemctl list-units --state=failed
```

### Systemd timer для мониторинга

**Файл:** `/etc/systemd/system/health-check.service`

```ini
[Unit]
Description=Health Check
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/health-check.sh
StandardOutput=journal
StandardError=journal
```

**Файл:** `/etc/systemd/system/health-check.timer`

```ini
[Unit]
Description=Health Check Timer
Requires=health-check.service

[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

**Активировать:**
```bash
sudo systemctl enable --now health-check.timer
journalctl -u health-check
```

---

## 🛡️ BEST PRACTICES

### 1. Чистить логи регулярно

```bash
# В crontab
0 3 * * * sudo journalctl --vacuum-time=7d
```

### 2. Мониторить важные метрики

```bash
# Диск (оповещение если > 90%)
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | cut -d'%' -f1)
if [ $DISK_USAGE -gt 90 ]; then
    echo "ALERT: Disk > 90%"
fi

# Memory (оповещение если > 80%)
MEM_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
if [ $MEM_USAGE -gt 80 ]; then
    echo "ALERT: Memory > 80%"
fi
```

### 3. Ограничить размер логов

**Файл:** `/etc/systemd/journald.conf`

```ini
[Journal]
SystemMaxUse=2G          # Максимум для всех логов
SystemMaxFileSize=100M   # Максимум на файл
RuntimeMaxUse=500M       # Для runtime logs
```

**Применить:**
```bash
sudo systemctl restart systemd-journald
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### Проблема: Логи занимают слишком много место

```bash
# Очистить старые
sudo journalctl --vacuum-time=3d

# Очистить до размера
sudo journalctl --vacuum-size=500M
```

### Проблема: Сервис медленно запускается

```bash
# Найти медленный сервис
systemd-analyze blame | head -10

# Подробнее
systemd-analyze critical-chain
```

---

## 📋 ШПАРГАЛКА МОНИТОРИНГА

```bash
# Логи
journalctl -f                    # Лайв
journalctl -u docker             # Сервис
journalctl --since "1 hour ago"  # По времени
sudo journalctl --vacuum-time=3d # Очистить

# Производительность
systemd-analyze                  # Загрузка
systemd-analyze blame            # По времени
ps aux --sort=-%cpu             # Top CPU
free -h                          # Memory
df -h                            # Disk

# Сервисы
systemctl list-units --type=service    # Все
systemctl list-units --state=failed    # Failed
systemctl list-timers                  # Таймеры
```

---

## ✅ ЗАВЕРШЕНО!

Вы прошли через все основные аспекты управления Linux системой через systemd.

→ Начните с [01-what-is-systemd.md](./01-what-is-systemd.md) для полного понимания!
