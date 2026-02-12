---
title: "Управление сервисами"
type: how-to
tags: [linux, systemd, systemctl, services, journalctl, enable, start, status]
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/systemd-reference]]"
  - "[[linux/explanation/process-model]]"
---

# Управление сервисами

> **TL;DR:** `systemctl start/stop/restart svc` — управление. `enable` — автозапуск при boot.
> `journalctl -u svc -f` — логи. Свои сервисы — в `/etc/systemd/system/`.

## Основные команды

```bash
# Управление
sudo systemctl start nginx         # запустить
sudo systemctl stop nginx          # остановить
sudo systemctl restart nginx       # перезапустить
sudo systemctl reload nginx        # перечитать конфиг (без остановки)

# Автозапуск
sudo systemctl enable nginx        # включить при boot
sudo systemctl disable nginx       # отключить при boot
sudo systemctl enable --now nginx  # enable + start одной командой

# Статус
systemctl status nginx             # подробный статус
systemctl is-active nginx          # active/inactive
systemctl is-enabled nginx         # enabled/disabled

# Списки
systemctl list-units --type=service           # запущенные
systemctl list-unit-files --type=service      # все (enabled/disabled)
systemctl --failed                             # проблемные
```

## Логи (journalctl)

```bash
journalctl -u nginx                # все логи сервиса
journalctl -u nginx -f             # follow (реальное время)
journalctl -u nginx --since "1h ago"
journalctl -u nginx -n 50          # последние 50 строк
journalctl -u nginx -p err         # только ошибки
```

## Создание своего сервиса

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=app
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload       # ОБЯЗАТЕЛЬНО после создания/изменения
sudo systemctl enable --now myapp
systemctl status myapp
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| «Сервис не перезапускается после редактирования» | `sudo systemctl daemon-reload` |
| `enable` но сервис не работает | `enable` ≠ `start`. Используйте `enable --now` |
| `ExecStart` не находит binary | Только абсолютные пути: `/usr/bin/node`, не `node` |
| Сервис в цикле перезапуска | `journalctl -u svc -n 50` → найти причину падения |