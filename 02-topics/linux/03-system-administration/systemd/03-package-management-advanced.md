---
created: 2026-01-06
updated: 2026-01-06
type: reference
---

# 03: Продвинутое управление пакетами

## 🎯 ПРОДВИНУТЫЙ PACKAGE MANAGEMENT

После базового `pacman -S` и `apt install` есть масса полезных команд.

---

## 🔍 ПОИСК И АНАЛИЗ

### Поиск зависимостей

```bash
# Arch: показать что зависит от пакета
pacman -Si package | grep "Depends On"

# Ubuntu/Debian: показать зависимости
apt show package | grep Depends
```

### Анализ размера

```bash
# Arch: размер установленных пакетов
pacman -Si $(pacman -Qq) | grep -E "^Name|^Installed Size"

# Ubuntu: размер пакета
apt show package | grep "Installed-Size"
```

### Поиск orphan пакетов

```bash
# Arch: найти неиспользуемые
pacman -Qdtq

# Ubuntu: найти неиспользуемые
sudo apt autoremove --dry-run
```

---

## 🔧 ADVANCED PACMAN (Arch)

### Конфликты пакетов

```bash
# Показать conflicting пакеты
pacman -Qu

# Форсировать установку (опасно!)
sudo pacman -S --overwrite='*' package

# Проверить целостность
sudo pacman -Dk
```

### Очистка cache

```bash
# Сохранить последние 3 версии
sudo paccache -rk 3

# Удалить весь cache
sudo pacman -Sc

# Удалить cache с unused пакетами
sudo pacman -Scc
```

---

## 🔧 ADVANCED APT (Ubuntu/Debian)

### Pinning (приоритеты пакетов)

**Файл:** `/etc/apt/preferences.d/myprefs`

```
Package: firefox
Pin: release a=unstable
Pin-Priority: 900
```

Теперь Firefox будет установлена из unstable, даже если есть stable версия.

### Версии и зависимости

```bash
# Показать все доступные версии
apt list -a mypackage

# Установить конкретную версию
sudo apt install mypackage=1.2.3

# Заморозить версию (не обновлять)
sudo apt-mark hold mypackage
sudo apt-mark unhold mypackage
```

---

## 🐳 СИСТЕМНЫЕ ПАКЕТЫ

### Настройка перед установкой

```bash
# dpkg-reconfigure (Ubuntu/Debian)
sudo dpkg-reconfigure package

# Пример: переконфигурация timezone
sudo dpkg-reconfigure tzdata
```

### Orphaned files

```bash
# Найти файлы от удаленных пакетов
sudo deborphan --all

# Удалить orphans
sudo deborphan --all | xargs sudo apt-get purge
```

---

## 📊 PACKAGE STATISTICS

### Статистика установки

```bash
# Arch: количество пакетов
pacman -Q | wc -l

# Ubuntu: количество пакетов
dpkg -l | wc -l

# Размер всех пакетов
pacman -Si $(pacman -Qq) | grep "Installed Size" | awk '{sum+=$4} END {print sum}'
```

---

## 🚨 ПРОБЛЕМЫ И РЕШЕНИЯ

### Проблема: Broken dependencies

```bash
# Arch: проверить
pacman -Dk

# Ubuntu: исправить
sudo apt --fix-broken install
```

### Проблема: Package conflicts

```bash
# Показать конфликтующие пакеты
pacman -Qu

# Ubuntu: исправить
sudo apt install -f
```

---

## 📋 ШПАРГАЛКА

```bash
# Поиск
pacman -Si package              # Информация (Arch)
apt show package                # Информация (Ubuntu)

# Зависимости
pacman -Sii package             # Вся цепь зависимостей
apt-cache depends package       # Зависимости

# Очистка
sudo paccache -rk 3             # Оставить 3 версии (Arch)
sudo apt autoremove             # Удалить orphans (Ubuntu)

# Версии
apt list -a package             # Все версии (Ubuntu)
sudo apt-mark hold package      # Заморозить (Ubuntu)
```

---

## 🔗 ДАЛЬШЕ

→ [04-backup-and-recovery.md](./04-backup-and-recovery.md)
