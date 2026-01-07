# Package Management Advanced

## NAME

Продвинутое управление пакетами: поиск, разрешение конфликтов, версионирование и оптимизация.

## SYNOPSIS

```bash
# Поиск
pacman -Si package                  # Информация (Arch)
apt show package                    # Информация (Ubuntu)

# Зависимости
pacman -Sii package                 # Вся цепь зависимостей (Arch)
apt-cache depends package           # Зависимости (Ubuntu)

# Версии
apt list -a package                 # Все версии (Ubuntu)
pacman -S package=version           # Конкретная версия (Arch)

# Очистка
sudo paccache -rk 3                 # Оставить 3 версии (Arch)
sudo apt autoremove                 # Удалить orphans (Ubuntu)
```

## DESCRIPTION

Администрирование пакетов на продвинутом уровне.

## SEARCHING AND ANALYSIS

### Поиск зависимостей

```bash
# Arch: показать что зависит от пакета
pacman -Si package | grep "Depends On"

# Ubuntu: показать зависимости
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

## ADVANCED PACMAN (Arch)

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

## ADVANCED APT (Ubuntu/Debian)

### Pinning (приоритеты пакетов)

**Файл:** `/etc/apt/preferences.d/myprefs`

```
Package: firefox
Pin: release a=unstable
Pin-Priority: 900
```

### Версии и зависимости

```bash
# Все доступные версии
apt list -a mypackage

# Конкретная версия
sudo apt install mypackage=1.2.3

# Заморозить версию
sudo apt-mark hold mypackage
sudo apt-mark unhold mypackage
```

## SYSTEM PACKAGES

### Настройка перед установкой

```bash
# dpkg-reconfigure (Ubuntu/Debian)
sudo dpkg-reconfigure package

# Переконфигурация timezone
sudo dpkg-reconfigure tzdata
```

### Orphaned files

```bash
# Найти файлы от удаленных пакетов
sudo deborphan --all

# Удалить orphans
sudo deborphan --all | xargs sudo apt-get purge
```

## PACKAGE STATISTICS

```bash
# Arch: количество пакетов
pacman -Q | wc -l

# Ubuntu: количество пакетов
dpkg -l | wc -l

# Размер всех пакетов (Arch)
pacman -Si $(pacman -Qq) | grep "Installed Size" | awk '{sum+=$4} END {print sum}'
```

## TROUBLESHOOTING

### Broken dependencies

```bash
# Arch: проверить
pacman -Dk

# Ubuntu: исправить
sudo apt --fix-broken install
```

### Package conflicts

```bash
# Показать конфликтующие пакеты (Arch)
pacman -Qu

# Ubuntu: исправить
sudo apt install -f
```

## KEY COMMANDS

```bash
pacman -Si package              # Информация (Arch)
apt show package                # Информация (Ubuntu)
pacman -Sii package             # Вся цепь зависимостей
apt-cache depends package       # Зависимости
sudo paccache -rk 3             # Оставить 3 версии (Arch)
sudo apt autoremove             # Удалить orphans (Ubuntu)
apt list -a package             # Все версии (Ubuntu)
sudo apt-mark hold package      # Заморозить (Ubuntu)
```

## KEY TAKEAWAYS

- **Поиск** — понять зависимости
- **Версии** — контролировать какую версию устанавливать
- **Очистка** — освобождать место
- **Конфликты** — разрешать правильно

## SEE ALSO

- [[./01-what-is-systemd.md|What is systemd]]
- [[./02-units-services.md|Units and Services]]
- [[./04-backup-and-recovery.md|Backup and Recovery]]
- [[./README.md|systemd README]]