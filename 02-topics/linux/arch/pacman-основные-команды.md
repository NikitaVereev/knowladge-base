---
created: 2026-01-01
tags: [arch-linux, pacman, tools]
type: reference
---

# Pacman основные команды

## Основная идея

Pacman - менеджер пакетов Arch Linux. Основной инструмент для установки, обновления и удаления пакетов.

## Детали

### Установка
```bash
sudo pacman -S package_name # Установить пакет
sudo pacman -S package1 package2 # Несколько пакетов
```

### Обновление системы
```bash
sudo pacman -Syu # Полное обновление
sudo pacman -Syyu # Принудительное обновление баз
```

### Удаление
```bash
sudo pacman -R package # Удалить пакет
sudo pacman -Rs package # + зависимости
sudo pacman -Rns package # + конфиги
```

### Поиск
```bash
pacman -Ss keyword # Поиск в репозиториях
pacman -Qs keyword # Поиск установленных
pacman -Qi package # Инфо об установленном
```

### Очистка кэша
```bash
sudo pacman -Sc # Удалить старые версии
sudo pacman -Scc # Полная очистка кэша
```

## Связанные заметки

- [[aur-yay-команды]]
- [[arch-maintenance]]
- [[arch-troubleshooting]]

## Источники

- Arch Wiki: Pacman
- man pacman
---
Создано: 2026-01-01 18:02
