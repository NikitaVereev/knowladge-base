---
created: 2026-01-02
tags: [linux, systemd, services, reference]
type: reference
---

# Systemd основы

## Основная идея

systemd - система инициализации Linux (PID 1). Запускается первым, управляет:
- Сервисами (приложения, демоны)
- Процессами и их зависимостями
- Логированием (journalctl)
- Ресурсами (CPU, память, файлы)
- Системными событиями (загрузка, выключение)

**Почему важно знать:**
- systemd на всех современных Linux дистрибутивах
- Неправильная конфигурация = проблемы в production
- journalctl дает полную видимость всех процессов

---

## ЧАСТЬ 1: Базовое управление сервисами

### Запуск и остановка

```bash
# Запустить сервис (сейчас, пока система работает)
sudo systemctl start nginx.service
# или без .service
sudo systemctl start nginx

# Остановить сервис
sudo systemctl stop nginx

# Перезапустить (стоп + старт)
sudo systemctl restart nginx

# Перезагрузить конфигурацию БЕЗ перезапуска
sudo systemctl reload nginx
# (если сервис поддерживает - иначе ошибка)

# Попробовать перезагрузить, если не получится - рестарт
sudo systemctl reload-or-restart nginx
```

**Разница между restart и reload:**
- `restart` = остановить + запустить заново (может быть downtime)
- `reload` = прочитать новую конфигурацию БЕЗ перезапуска (no downtime, если сервис поддерживает)

### Автозапуск при загрузке

```bash
# Включить автозапуск (сервис будет стартовать при загрузке)
sudo systemctl enable nginx

# Отключить автозапуск (не будет запускаться при загрузке)
sudo systemctl disable nginx

# Проверить включен ли автозапуск
systemctl is-enabled nginx
# Вывод: enabled или disabled

# Включить И запустить сразу (две операции вместе)
sudo systemctl enable --now nginx

# Отключить И остановить сразу
sudo systemctl disable --now nginx
```

**Что происходит при enable:**
- systemd создает символическую ссылку в `/etc/systemd/system/multi-user.target.wants/`
- При загрузке systemd видит эту ссылку и запускает сервис

### Проверка статуса

```bash
# Подробный статус (САМАЯ ПОЛЕЗНАЯ КОМАНДА!)
systemctl status nginx
```

Вывод выглядит так:
```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since Fri 2026-01-02 10:00:00 MSK; 2h 30min ago
   Process: 12345 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
    Main PID: 12346 (nginx)
     CGroup: /system.slice/nginx.service
             ├─12346 nginx: master process /usr/sbin/nginx
             └─12347 nginx: worker process

Jan 02 10:00:00 myhost systemd[1]: Started A high performance web server...
```

**Что смотреть:**
- `Loaded:` - конфиг загружен? (loaded = да)
- `enabled` - будет ли запускаться при загрузке?
- `Active:` - запущен сейчас? (active (running) = да)
- `Main PID:` - ID главного процесса
- Последние логи сервиса

### Список всех сервисов

```bash
# Список АКТИВНЫХ юнитов
systemctl list-units --type service
# Покажет только запущенные сервисы

# Список ВСЕ доступных сервис-файлов (включая отключенные)
systemctl list-unit-files --type service
# Покажет все .service файлы с их статусом

# Упавшие сервисы (которые не запустились или упали)
systemctl --failed
# СМОТРИТЕ СЮДА если что-то не работает!
```

---

## ЧАСТЬ 2: Логирование (journalctl)

systemd логирует ВСЕ в журнал journalctl. Это лучше чем `/var/log/` файлы.

### Основные команды

```bash
# Логи конкретного сервиса (все строки)
journalctl -u nginx

# Real-time логи (как tail -f)
journalctl -u nginx -f

# Последние N строк
journalctl -u nginx -n 50

# За последний час
journalctl -u nginx --since "1 hour ago"

# За последние 24 часа
journalctl -u nginx --since "1 day ago"

# Между двумя временами
journalctl -u nginx --since "2026-01-01 10:00:00" --until "2026-01-02 10:00:00"

# С текущей загрузки системы
journalctl -u nginx -b

# С предыдущей загрузки
journalctl -u nginx -b -1

# Только ошибки
journalctl -u nginx -p err

# Только предупреждения и ошибки
journalctl -u nginx -p warn

# По приоритету: emerg, alert, crit, err, warning, notice, info, debug
journalctl -p crit  # Критические ошибки системы

# Без pager'а (без возможности скроллить, просто вывод)
journalctl -u nginx --no-pager

# Экспортировать в JSON (для парсинга)
journalctl -u nginx -o json
```

### Быстрые примеры

```bash
# Посмотреть почему сервис не запустился (последние ошибки)
journalctl -u myapp -n 50 -p err

# Следить за сервисом в реальном времени
journalctl -u myapp -f

# Все логи ошибок системы за последний час
journalctl -p err --since "1 hour ago"

# Поиск по тексту (просто вывести и grep)
journalctl -u nginx | grep "error"

# Количество строк логов
journalctl -u nginx | wc -l
```

---

## ЧАСТЬ 3: Анализ системы и загрузки

### Время загрузки

```bash
# Общее время загрузки
systemd-analyze
# Вывод:
# Startup finished in 2.345s (kernel) + 1.234s (userspace) = 3.579s

# Что тормозит (отсортировано по времени)
systemd-analyze blame
# Покажет какие сервисы долго загружаются

# Критический путь загрузки (зависимости)
systemd-analyze critical-chain
# Покажет цепочку: network → postgresql → myapp → etc
```

### Проверка конфигурации

```bash
# Проверить синтаксис .service файла
systemd-analyze verify myapp.service
# Покажет все ошибки синтаксиса

# Security анализ (насколько безопасна конфигурация)
systemd-analyze security myapp.service
# Выдаст рекомендации по улучшению security
```

### Зависимости сервисов

```bash
# Что нужно для этого сервиса
systemctl list-dependencies nginx
# Покажет: network, system mounts, etc

# Кто зависит от этого сервиса (обратная зависимость)
systemctl list-dependencies --reverse nginx
# Обычно пусто если это не базовый сервис
```

---

## ЧАСТЬ 4: Управление системой (перезагрузка, выключение)

### Перезагрузка и выключение

```bash
# Перезагрузить систему
sudo systemctl reboot
# или короче:
sudo reboot

# Выключить систему
sudo systemctl poweroff
# или короче:
sudo poweroff

# Остановить систему (но не выключать питание)
sudo systemctl halt
# или короче:
sudo halt

# Режим сна
sudo systemctl suspend

# Гибернация (сохранить в файл и выключить)
sudo systemctl hibernate

# Гибернация + сон вместе
sudo systemctl hybrid-sleep
```

### Отложенная перезагрузка

```bash
# Перезагрузить через 10 минут (обратный отсчет в консоли)
sudo shutdown -r +10

# Выключить через 1 час
sudo shutdown -h +60

# Выключить в конкретное время (17:30)
sudo shutdown -h 17:30

# Отменить отложенное выключение
sudo shutdown -c
```

---

## ЧАСТЬ 5: Создание простого сервиса (быстрый старт)

### Минимальный пример: Node.js приложение

**Файл:** `/etc/systemd/system/myapp.service`

```ini
[Unit]
Description=My Node.js Application
After=network.target

[Service]
Type=simple
User=myuser
ExecStart=/usr/bin/node /opt/myapp/app.js
Restart=on-failure
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Установить:**
```bash
# 1. Создать файл (копировать текст выше)
sudo nano /etc/systemd/system/myapp.service

# 2. Проверить синтаксис
sudo systemd-analyze verify myapp.service

# 3. Загрузить в systemd
sudo systemctl daemon-reload

# 4. Включить при загрузке и запустить
sudo systemctl enable --now myapp

# 5. Проверить
sudo systemctl status myapp
sudo journalctl -u myapp -f
```

### Python Flask пример

```ini
[Unit]
Description=My Flask App
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/flask-app
ExecStart=/usr/bin/python3 /opt/flask-app/app.py
Restart=on-failure
Environment="FLASK_ENV=production"

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Nginx пример (уже существует, но вот как выглядит)

```ini
[Unit]
Description=Nginx HTTP and reverse proxy server
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

---

## ЧАСТЬ 6: Типичные проблемы и их решение

### "Unit not found"
```bash
systemctl status myapp
# Unit myapp.service could not be found.

# Решение:
# 1. Проверить что файл создан в правильном месте
ls -la /etc/systemd/system/myapp.service

# 2. Загрузить конфиг
sudo systemctl daemon-reload

# 3. Попробовать снова
systemctl status myapp
```

### "Failed to start" или сервис упал сразу

```bash
# 1. Посмотреть ошибку в логах
sudo journalctl -u myapp -n 50

# 2. Проверить что ExecStart правильный
which myapp              # убедиться что программа существует
/path/to/myapp --help   # протестировать команду вручную

# 3. Если ExecStart неправильный - отредактировать
sudo systemctl edit myapp.service
# или
sudo nano /etc/systemd/system/myapp.service

# 4. Загрузить и перезапустить
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

### "Permission denied"

```bash
# Обычно когда файл приложения недоступен для пользователя
# Решение:

# 1. Проверить права доступа
ls -la /opt/myapp/app.js

# 2. Если прав нет - выдать права пользователю
sudo chown myuser:myuser /opt/myapp -R
sudo chmod u+x /opt/myapp/app.js

# 3. Или запускать под пользователем который может читать файлы
sudo systemctl edit myapp.service
# Изменить: User=root (НЕ рекомендуется)
```

### Сервис перезапускается в loop

```bash
# Симптомы: systemctl status показывает что сервис падает и перезапускается постоянно

# Решение:
# 1. Посмотреть логи
sudo journalctl -u myapp -f

# 2. Обычно есть error в логах - исправить в приложении

# 3. Если хотите отключить авто-перезапуск временно:
sudo systemctl edit myapp.service
# Изменить: Restart=no
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

---

## ЧАСТЬ 7: Шпаргалка (быстрый справочник)

### Основные команды

```bash
# УПРАВЛЕНИЕ
sudo systemctl start service          # Запустить
sudo systemctl stop service           # Остановить
sudo systemctl restart service        # Перезапустить
sudo systemctl reload service         # Перечитать конфиг
sudo systemctl reload-or-restart service

# АВТОЗАПУСК
sudo systemctl enable service         # Включить
sudo systemctl disable service        # Отключить
systemctl is-enabled service          # Проверить

# СТАТУС
systemctl status service              # Полный статус
systemctl list-units --type service   # Все активные
systemctl --failed                    # Упавшие
systemctl list-unit-files --type service

# ЛОГИ
journalctl -u service                 # Все логи
journalctl -u service -f              # Real-time
journalctl -u service -n 50           # Последние 50
journalctl -u service -p err          # Только ошибки
journalctl -u service -b              # С текущей загрузки
journalctl -u service --since "1 hour ago"

# АНАЛИЗ
systemd-analyze                       # Время загрузки
systemd-analyze blame                 # Что тормозит
systemd-analyze verify service.service
systemd-analyze security service.service

# РЕДАКТИРОВАНИЕ (БЕЗ SUDO!)
sudo systemctl edit service           # Отредактировать конфиг

# СИСТЕМА
sudo systemctl reboot                 # Перезагрузка
sudo systemctl poweroff               # Выключение
sudo systemctl suspend                # Сон
```

### Шаблон для проверки проблем

```bash
# Если сервис не работает - выполни по порядку:

# 1. Статус
sudo systemctl status myapp

# 2. Логи (50 последних строк)
sudo journalctl -u myapp -n 50

# 3. Логи (real-time)
sudo journalctl -u myapp -f

# 4. Проверить конфиг
sudo systemd-analyze verify myapp.service

# 5. Проверить что приложение работает вручную
/opt/myapp/myapp --help

# 6. Перезагрузить конфиг и перезапустить
sudo systemctl daemon-reload
sudo systemctl restart myapp

# 7. Повторить шаг 2
sudo journalctl -u myapp -n 50
```

---

## ЧАСТЬ 8: Полезные факты

### Где находятся сервисы

```bash
# Системные сервисы (от дистрибутива)
/usr/lib/systemd/system/
# Примеры: nginx.service, postgresql.service, docker.service

# Ваши сервисы (которые вы создали)
/etc/systemd/system/
# Эти переписывают системные

# Пользовательские сервисы (без sudo, только для вас)
~/.config/systemd/user/
```

### Создание симлинков для удобства

```bash
# Если часто используете длинное имя
sudo systemctl edit --full myapp.service  # Вместо nano

# Или создать alias в ~/.bashrc
alias restart-myapp="sudo systemctl restart myapp"
alias logs-myapp="sudo journalctl -u myapp -f"
```

### Performance tips

```bash
# Если система медленно загружается
systemd-analyze blame | head -10   # Посмотреть top 10

# Если сервис стартует медленно
systemd-analyze verify myapp.service
# Может быть ненужная зависимость (After=...)

# Ограничение памяти/CPU
sudo systemctl edit myapp.service
# Добавить:
# [Service]
# MemoryMax=512M
# CPUQuota=50%
```

---

## Связанные заметки

- [[systemd-service-creation]] - как создать свой сервис
- [[systemd-timers]] - расписание вместо cron
- [[linux-troubleshooting]] - общие проблемы Linux
- [[arch-maintenance]] - обслуживание Arch Linux

## Источники

- `man systemctl` - полная документация
- `man journalctl` - документация логов
- `man systemd.unit` - формат unit файлов
- Arch Wiki: systemd
- Freedesktop.org systemd documentation

---

Создано: 2026-01-02 07:02
