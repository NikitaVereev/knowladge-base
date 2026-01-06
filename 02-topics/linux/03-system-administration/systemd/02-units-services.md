---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# 02: Service файлы и управление сервисами

## 🎯 ЧТО ТАКОЕ SERVICE ФАЙЛЫ?

Service файлы описывают как systemd должен управлять сервисом.

**Расположение:**
```bash
/etc/systemd/system/              # Пользовательские/локальные
/usr/lib/systemd/system/          # Системные встроенные
/run/systemd/system/              # Runtime
```

---

## 📝 БАЗОВЫЙ SERVICE ФАЙЛ

**Файл:** `/etc/systemd/system/hello.service`

```ini
[Unit]
Description=Hello Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/echo "Hello World"
RemainAfterExit=no

[Install]
WantedBy=multi-user.target
```

**Активировать:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable hello.service
sudo systemctl start hello.service
journalctl -u hello.service
```

---

## 🔧 ТИПЫ SERVICE

### simple (по умолчанию)

```ini
[Service]
Type=simple
ExecStart=/usr/bin/myapp
```

Процесс остается работающим в foreground.

### oneshot

```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

Выполнить один раз и выйти. Полезно для скриптов.

### forking

```ini
[Service]
Type=forking
ExecStart=/usr/bin/legacy-daemon
PIDFile=/var/run/myapp.pid
```

Для старых daemon'ов что fork.

### dbus

```ini
[Service]
Type=dbus
BusName=com.example.myservice
ExecStart=/usr/bin/myapp
```

Для D-Bus сервисов.

---

## 🛡️ SERVICE SANDBOXING (БЕЗОПАСНОСТЬ)

### Важные опции:

```ini
[Service]
# Часы и время (не трогать)
ProtectClock=yes

# Не создавать исполняемую память
MemoryDenyWriteExecute=yes

# Не создавать user namespaces
RestrictNamespaces=~user

# Не создавать process namespaces
RestrictNamespaces=~pid

# No new privileges
NoNewPrivileges=yes

# Приватная tmp папка
PrivateTmp=yes
```

### Пример БЕЗОПАСНОГО сервиса:

```ini
[Unit]
Description=My Secure Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/myservice
User=myservice
Group=myservice

# БЕЗОПАСНОСТЬ
ProtectClock=yes
MemoryDenyWriteExecute=yes
RestrictNamespaces=~user
NoNewPrivileges=yes
PrivateTmp=yes

# ОГРАНИЧЕНИЕ РЕСУРСОВ
MemoryLimit=512M
CPUQuota=50%

[Install]
WantedBy=multi-user.target
```

---

## ⏲️ SYSTEMD TIMERS (ВМЕСТО CRON)

### Создать backup таймер

**Файл:** `/etc/systemd/system/backup.service`

```ini
[Unit]
Description=Daily Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
```

**Файл:** `/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Daily Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Активировать:**
```bash
sudo systemctl enable --now backup.timer
systemctl status backup.timer
journalctl -u backup.service
```

---

## 📋 ОСНОВНЫЕ КОМАНДЫ

```bash
# Перезагрузить конфиги
sudo systemctl daemon-reload

# Перезагрузить один сервис
sudo systemctl reload docker

# Синтаксис проверка
sudo systemd-analyze verify /etc/systemd/system/myservice.service

# Зависимости
systemctl list-dependencies docker

# Требования перед запуском
systemctl list-unit-files --type=service
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### Проблема: Сервис не запускается

```bash
# Посмотреть ошибку
systemctl status myservice
journalctl -u myservice -e

# Проверить синтаксис
sudo systemd-analyze verify /etc/systemd/system/myservice.service

# Перезагрузить конфиги
sudo systemctl daemon-reload
```

### Проблема: Сервис зависает

```bash
# Принудительно остановить
sudo systemctl kill myservice

# С SIGKILL сразу
sudo systemctl kill -9 myservice
```

---

## 📋 ШПАРГАЛКА

```bash
# Управление
sudo systemctl daemon-reload      # Перезагрузить конфиги
sudo systemctl start docker       # Запустить
sudo systemctl stop docker        # Остановить
sudo systemctl restart docker     # Перезагрузить
sudo systemctl reload docker      # Перезагрузить без restart
sudo systemctl enable docker      # Автозагрузка
sudo systemctl disable docker     # Отключить

# Таймеры
systemctl list-timers             # Список
sudo systemctl enable --now backup.timer  # Создать

# Проверка
sudo systemd-analyze verify file.service  # Синтаксис
systemctl status docker            # Статус
```

---

## 🔗 ДАЛЬШЕ

→ [03-package-management-advanced.md](./03-package-management-advanced.md)
