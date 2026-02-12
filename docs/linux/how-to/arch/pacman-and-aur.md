---
title: "pacman и AUR"
type: how-to
tags: [linux, arch, pacman, aur, yay, paru, packages, mirrors]
sources:
  original: "_inbox/01-linux/02-distro-specific/arch-linux/02-pacman-guide.md + 03-aur-guide.md"
related:
  - "[[linux/how-to/arch/install]]"
  - "[[linux/how-to/arch/maintenance]]"
  - "[[linux/tutorials/02-package-management]]"
---

# pacman и AUR

> **TL;DR:** `pacman -Syu` — обновить всё. `pacman -S pkg` — установить. `pacman -Rs pkg` — удалить.
> AUR — пользовательский репозиторий. Использовать yay/paru как обёртку.

## pacman — основные команды

### Синхронизация и обновление

```bash
sudo pacman -Syu                   # обновить базу + обновить все пакеты
sudo pacman -Sy                    # только обновить базу (⚠️ partial upgrade — опасно!)
sudo pacman -Syyu                  # принудительно обновить базу + пакеты
```

> **Важно:** Никогда не делайте `pacman -Sy package`. Всегда `pacman -Syu` перед установкой. Partial upgrade ломает систему.

### Установка и удаление

```bash
sudo pacman -S nginx               # установить
sudo pacman -S nginx vim git       # несколько пакетов
sudo pacman -R nginx               # удалить (оставить зависимости)
sudo pacman -Rs nginx              # удалить + неиспользуемые зависимости
sudo pacman -Rns nginx             # удалить + зависимости + конфиги
```

### Поиск и информация

```bash
pacman -Ss nginx                   # найти в репозиториях
pacman -Si nginx                   # информация (из repo)
pacman -Qi nginx                   # информация (установленный)
pacman -Ql nginx                   # список файлов пакета
pacman -Qo /usr/bin/nginx          # какому пакету принадлежит файл?
pacman -Qdt                        # orphan-пакеты (никто не зависит)
pacman -Q                          # все установленные
pacman -Qe                         # только явно установленные
```

### Очистка

```bash
sudo pacman -Sc                    # удалить кэш старых версий
sudo pacman -Scc                   # удалить ВЕСЬ кэш
sudo pacman -Rns $(pacman -Qdtq)  # удалить orphans
```

### Конфигурация

```bash
# /etc/pacman.conf — основной конфиг
# Раскомментировать для параллельной загрузки:
# ParallelDownloads = 5

# Раскомментировать для цветного вывода:
# Color

# /etc/pacman.d/mirrorlist — зеркала
# Обновить зеркала:
sudo reflector --country "Germany" --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

## AUR (Arch User Repository)

AUR — пользовательский репозиторий с PKGBUILD-скриптами. Содержит ~90000 пакетов, которых нет в официальных репозиториях.

### yay (рекомендуемый AUR helper)

```bash
# Установка yay
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si

# Использование (синтаксис как у pacman)
yay -S google-chrome               # установить из AUR
yay -Syu                           # обновить всё (repo + AUR)
yay -Ss telegram                   # поиск (repo + AUR)
yay -Rs package                    # удалить
yay                                # интерактивное обновление
```

### paru (альтернатива yay, на Rust)

```bash
# Установка
yay -S paru
# или
git clone https://aur.archlinux.org/paru.git
cd paru && makepkg -si

# Использование идентично yay
paru -S package
paru -Syu
```

### Ручная установка из AUR

```bash
# 1. Клонировать PKGBUILD
git clone https://aur.archlinux.org/package-name.git
cd package-name

# 2. Проверить PKGBUILD (ВАЖНО! Читайте что он делает)
cat PKGBUILD

# 3. Собрать и установить
makepkg -si
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `error: failed to commit transaction (conflicting files)` | `pacman -S --overwrite '*' package` или удалить конфликтный файл |
| Partial upgrade (`-Sy pkg`) | Никогда! Всегда `pacman -Syu` |
| `PGP signature error` | `sudo pacman-key --refresh-keys` |
| AUR пакет не собирается | Прочитать комментарии на AUR-странице пакета |
| Зеркала медленные | `sudo reflector --sort rate --save /etc/pacman.d/mirrorlist` |