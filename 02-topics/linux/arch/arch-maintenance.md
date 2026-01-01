---
created: 2026-01-01
tags: []
type: note
---

# Arch maintenance регулярные задачи

## Основная идея

Список операций для поддержания Arch в здоровом состоянии.

## Детали

### Еженедельно

- `sudo pacman -Syu` - обновить систему
- `yay -Yc` - очистить orphans
- Проверить логи: `journalctl -p 3 -xb`

### Ежемесячно

- `sudo pacman -Sc` - очистить кэш пакетов
- Проверить failed services: `systemctl --failed`
- Бэкап критичных конфигов

## Связанные заметки

- [[pacman-basic-commands]]
- [[aur-yay-commands]]
- [[system-backup-strategy]]

## Источники

---
Создано: 2026-01-01 18:29
