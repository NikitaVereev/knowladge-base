# Units and Services

## NAME

systemd Units: конфигурационные файлы для управления сервисами и ресурсами.

## SYNOPSIS

```bash
sudo nano /etc/systemd/system/myservice.service
sudo systemctl daemon-reload
sudo systemctl start myservice
sudo systemctl enable myservice
journalctl -u myservice -f
```

## DESCRIPTION

Units — это конфигурационные файлы в формате .ini которые определяют как systemd управляет ресурсами.

## SERVICE FILES

### Структура

```ini
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp
Restart=always
User=myuser

[Install]
WantedBy=multi-user.target
```

### [Unit] раздел

```
Description        — описание сервиса
After              — запускать после (порядок)
Before             — запускать до
Wants              — хочет но не требует
Requires           — требует (критично)
Conflicts          — конфликтует с
```

### [Service] раздел

```
Type               — simple (по умолчанию), forking, oneshot
ExecStart          — команда запуска (обязателен)
ExecStop           — команда остановки
ExecStartPre       — перед запуском
ExecStartPost      — после запуска

Restart            — always, on-failure, no
RestartSec         — ждать перед перезапуском

User               — пользователь
Group              — группа
WorkingDirectory   — рабочая директория
Environment        — переменные окружения

StandardOutput     — journal, syslog, kmsg, null
StandardError      — journal, syslog, kmsg, null
```

### [Install] раздел

```
WantedBy           — включить в этот target
RequiredBy         — требуется для другого target
Alias              — альтернативное имя
```

## EXAMPLES

### Пример 1: Простой сервис

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myuser
WorkingDirectory=/home/myuser/app
ExecStart=/home/myuser/app/start.sh
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Пример 2: Python Web Server

```ini
[Unit]
Description=Python Web App
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /srv/app/main.py
WorkingDirectory=/srv/app
User=www-data
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal

PrivateTmp=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

### Пример 3: Backup Timer

**backup.timer:**
```ini
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

**backup.service:**
```ini
[Unit]
Description=Backup Script

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup

[Install]
WantedBy=multi-user.target
```

## MANAGING SERVICES

### systemctl команды

```bash
systemctl start service              # запустить
systemctl stop service               # остановить
systemctl restart service            # перезагрузить
systemctl reload service             # перезагрузить конфиг

systemctl enable service             # включить при загрузке
systemctl disable service            # отключить при загрузке

systemctl status service             # статус
systemctl is-active service          # активен ли

systemctl list-units                 # все units
systemctl list-unit-files            # все .service файлы
systemctl list-dependencies service  # зависимости

systemctl daemon-reload              # перезагрузить конфиги (ОБЯЗАТЕЛЕН)
```

### journalctl логирование

```bash
journalctl -u service                # логи сервиса
journalctl -u service -f              # в реальном времени
journalctl -u service -n 50           # последние 50 строк
journalctl -u service -p err          # только ошибки
journalctl -u service --since "1 hour ago"  # за час
```

## CREATING A SERVICE

1. Создайте файл:
```bash
sudo nano /etc/systemd/system/myservice.service
```

2. Добавьте содержимое (используйте примеры выше)

3. Перезагрузите конфиги:
```bash
sudo systemctl daemon-reload
```

4. Включите и запустите:
```bash
sudo systemctl enable myservice
sudo systemctl start myservice
```

5. Проверьте:
```bash
sudo systemctl status myservice
journalctl -u myservice -f
```

## TROUBLESHOOTING

### Сервис не запускается

```bash
sudo systemctl status myservice
journalctl -u myservice -n 50
systemd-analyze verify /etc/systemd/system/myservice.service
```

### Сервис зависает

```bash
sudo systemctl stop myservice
sudo systemctl daemon-reload
sudo systemctl start myservice
journalctl -u myservice -f
```

## KEY TAKEAWAYS

- **.service файлы** — в /etc/systemd/system/
- **systemctl** — управление сервисами
- **daemon-reload** — обязателен после изменений
- **journalctl** — просмотр логов
- **Type=simple** — обычный сервис

## SEE ALSO

- [[./01-what-is-systemd.md|What is systemd]]
- [[./03-package-management-advanced.md|Package Management]]
- [[./README.md|systemd README]]