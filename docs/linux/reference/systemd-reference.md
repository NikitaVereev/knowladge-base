---
title: "Справочник: systemd"
type: reference
tags: [linux, systemd, systemctl, journalctl, reference, timers, units]
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/how-to/manage-services]]"
  - "[[linux/reference/cheatsheet]]"
---

# Справочник: systemd

## systemctl

| Команда | Действие |
|---------|---------|
| `systemctl start svc` | Запустить |
| `systemctl stop svc` | Остановить |
| `systemctl restart svc` | Перезапустить |
| `systemctl reload svc` | Перечитать конфиг (без остановки) |
| `systemctl enable svc` | Включить автозапуск |
| `systemctl disable svc` | Отключить автозапуск |
| `systemctl enable --now svc` | Enable + start |
| `systemctl status svc` | Статус и последние логи |
| `systemctl is-active svc` | active/inactive |
| `systemctl is-enabled svc` | enabled/disabled |
| `systemctl daemon-reload` | Перечитать unit-файлы (**после изменений!**) |
| `systemctl list-units --type=service` | Запущенные сервисы |
| `systemctl list-unit-files` | Все сервисы (enabled/disabled) |
| `systemctl --failed` | Сломанные |
| `systemctl mask svc` | Полностью заблокировать |
| `systemctl unmask svc` | Разблокировать |

## journalctl

| Команда | Действие |
|---------|---------|
| `journalctl -u svc` | Логи сервиса |
| `journalctl -u svc -f` | Follow (реальное время) |
| `journalctl -u svc -n 50` | Последние 50 строк |
| `journalctl -u svc -p err` | Только ошибки |
| `journalctl -b` | С последней загрузки |
| `journalctl -b -1` | С предпоследней загрузки |
| `journalctl --since "1h ago"` | За последний час |
| `journalctl --since today` | За сегодня |
| `journalctl --disk-usage` | Размер журнала |
| `journalctl --vacuum-size=500M` | Ограничить размер |
| `journalctl --vacuum-time=2weeks` | Удалить старше 2 недель |

## Приоритеты (journalctl -p)

| Приоритет | Уровень | Описание |
|-----------|---------|---------|
| 0 | emerg | Система неработоспособна |
| 1 | alert | Требуется немедленное действие |
| 2 | crit | Критическая ошибка |
| 3 | err | Ошибка |
| 4 | warning | Предупреждение |
| 5 | notice | Важное уведомление |
| 6 | info | Информация |
| 7 | debug | Отладка |

## Шаблон service-файла

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target
Wants=postgresql.service

[Service]
Type=simple                    # simple | forking | oneshot | notify
User=app
Group=app
WorkingDirectory=/opt/myapp
Environment=NODE_ENV=production
EnvironmentFile=/opt/myapp/.env
ExecStart=/usr/bin/node server.js
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure             # on-failure | always | on-abnormal | no
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## Шаблон timer-файла

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=*-*-* 03:00:00     # ежедневно в 3:00
# OnCalendar=Mon *-*-* 03:00  # каждый понедельник
# OnBootSec=5min               # через 5 минут после загрузки
# OnUnitActiveSec=1h           # каждый час
Persistent=true                # запустить если пропустили

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now backup.timer
systemctl list-timers
```

## Расположение файлов

| Путь | Описание |
|------|---------|
| `/etc/systemd/system/` | Ваши юниты (высший приоритет) |
| `/run/systemd/system/` | Runtime (временные) |
| `/usr/lib/systemd/system/` | Юниты из пакетов (не редактировать!) |

## Типы сервисов (Type=)

| Type | Описание |
|------|---------|
| `simple` | ExecStart — основной процесс (default) |
| `forking` | Процесс форкается в фон (как nginx) |
| `oneshot` | Запускается и завершается (скрипты) |
| `notify` | Сервис сообщает о готовности через sd_notify |