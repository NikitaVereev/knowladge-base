---
created: 2026-01-01
tags: [arch-linux, aur, yay]
type: note
---

# AUR и yay команды

## Основная идея

AUR (Arch User Repository) - пользовательские пакеты.
yay - AUR helper для удобной установки.

## Детали

### Установка yay
```bash
# Установить git и base-devel
sudo pacman -S --needed git base-devel

# Клонировать репозиторий yay
git clone https://aur.archlinux.org/yay.git
cd yay

# Собрать и установить
makepkg -si

# Удалить папку после установки
cd ..
rm -rf yay
```

### Основные команды
```bash
yay package_name # Установить из AUR
yay -Syu # Обновить всё (+AUR)
yay -Yc # Очистить ненужные зависимости
yay -Ps # Статистика системы
```

### Поиск и информация
```bash
yay -Ss keyword # Поиск в AUR
yay -Si package # Инфо о пакете из AUR
yay -Qi package # Инфо об установленном
```

### Полезные флаги
```bash
yay -Sua # Обновить только AUR пакеты
yay -Qm # Список установленных из AUR
yay -c # Удалить ненужные зависимости
```

## Связанные заметки

## Источники

---
Создано: 2026-01-01 18:19
