---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# 01: Что такое systemd?

## 🎯 ЧТО ТАКОЕ systemd?

**systemd** — система инициализации и управления сервисами в современном Linux.

- **Инициализация** — запуск системы (PID 1)
- **Управление сервисами** — запуск/остановка/перезагрузка сервисов
- **Логирование** — journalctl для просмотра логов
- **Таймеры** — systemd timers (вместо cron)
- **Сетевые утилиты** — systemd-networkd, systemd-resolved
- **Контейнеры** — systemd-nspawn

---

## 🎯 ОСН ОСНОВНЫЕ КОНЦЕПЦИИ

### Units (юниты)

Всё в systemd — это units. Типы:

```
.service   — сервис (демон)
.socket    — сокет
.timer     — таймер (вместо cron)
.mount     — монтирование
.swap      — swap файл
.target    — группа юнитов
.device    — устройство
```

### Targets (целевые состояния)

```
multi-user.target    — command line (обычный режим)
graphical.target     — GUI режим
rescue.target        — single user mode
poweroff.target      — выключение
reboot.target        — перезагрузка
```

### States (состояния)

```
started              — работает
stopped              — остановлен
failed               — ошибка
active               — активен
inactive             — неактивен
```

---

## 📋 ОСНОВНЫЕ КОМАНДЫ

### Сервисы

```bash
# Статус сервиса
systemctl status docker

# Запустить сервис
sudo systemctl start docker

# Остановить
sudo systemctl stop docker

# Перезагрузить конфиг без перезагрузки
sudo systemctl reload docker

# Перезагрузить (stop + start)
sudo systemctl restart docker

# Автозагрузка при старте
sudo systemctl enable docker

# Отключить автозагрузку
sudo systemctl disable docker

# Список всех сервисов
systemctl list-units --type=service

# Список failed сервисов
systemctl list-units --state=failed
```

### Логирование (journalctl)

```bash
# Последние 50 строк
journalctl -n 50

# Лайв логи
journalctl -f

# Логи сегодня
journalctl --since today

# Логи конкретного сервиса
journalctl -u docker

# Логи с ошибками
journalctl -u docker --priority=err

# ✅ АКТУАЛЬНО: Очистить старые логи
sudo journalctl --vacuum-time=3d

# ❌ СТАРАЯ КОМАНДА (не работает)
sudo journalctl --vacuum=3d
```

### Таймеры

```bash
# Список всех таймеров
systemctl list-timers

# Активировать таймер
sudo systemctl enable --now backup.timer

# Логи таймера
journalctl -u backup.timer
```

---

## 📈 СИСТЕМНАЯ ИЕРАРХИЯ

```
/etc/systemd/              — конфигурация
├── system/                — системные юниты
├── user/                  — пользовательские юниты
├── journal-remote.conf    — remote logging
├── journald.conf          — логирование
└── timesyncd.conf         — время

/run/systemd/              — runtime данные
├── journal/               — логи текущей сессии
└── seats/                 — seats управление

/usr/lib/systemd/          — системные default юниты
└── system/                — встроенные сервисы
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### Проблема: Сервис не запускается

```bash
# Проверить ошибку
systemctl status myservice
journalctl -u myservice -e

# Проверить синтаксис файла
sudo systemd-analyze verify /etc/systemd/system/myservice.service
```

### Проблема: Забыли какой таймер нужен

```bash
# Показать все таймеры
systemctl list-timers

# Показать historyu таймера
journalctl -u myservice.timer
```

### Проблема: systemd занимает много памяти

```bash
# Ограничить журналы
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=100M
```

---

## 📋 ШПАРГАЛКА

```bash
# Основное
systemctl status docker           # Статус
sudo systemctl start docker       # Запустить
sudo systemctl stop docker        # Остановить
sudo systemctl restart docker     # Перезагрузить
sudo systemctl enable docker      # Автозагрузка

# Логи
journalctl -u docker             # Логи сервиса
journalctl -f                    # Лайв
sudo journalctl --vacuum-time=3d # Очистить

# Таймеры
systemctl list-timers            # Список
sudo systemctl enable --now timer # Создать
```

---

## 🔗 ДАЛЬШЕ

→ [02-units-services.md](./02-units-services.md) — Service файлы и управление
