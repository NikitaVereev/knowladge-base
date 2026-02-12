---
title: "Продвинутое управление пакетами"
type: how-to
tags: [linux, packages, advanced, dependencies, orphans, pinning, cache, source]
sources:
  original: "_inbox/01-linux/03-system-administration/systemd/03-package-management-advanced.md"
related:
  - "[[linux/tutorials/02-package-management]]"
  - "[[linux/how-to/arch/pacman-and-aur]]"
  - "[[linux/how-to/ubuntu/apt-and-ppa]]"
---

# Продвинутое управление пакетами

> **TL;DR:** Анализ зависимостей, поиск orphans, очистка кэша, pinning версий.
> Работает кроссплатформенно (apt, pacman, dnf с разным синтаксисом).

## Анализ зависимостей

```bash
# Что зависит от пакета?
# apt
apt rdepends nginx

# pacman
pactree -r nginx

# dnf
dnf repoquery --whatrequires nginx
```

```bash
# От чего зависит пакет?
# apt
apt depends nginx

# pacman
pactree nginx

# dnf
dnf repoquery --requires nginx
```

## Orphan-пакеты (ничьи зависимости)

```bash
# Найти orphans
# pacman
pacman -Qdtq

# Удалить orphans
# pacman
sudo pacman -Rns $(pacman -Qdtq)

# apt
sudo apt autoremove --purge

# dnf
sudo dnf autoremove
```

## Анализ размера

```bash
# Самые большие пакеты
# pacman
pacman -Qi | awk '/^Name/{name=$3} /^Installed Size/{print $4,$5,name}' | sort -rh | head -20

# apt (dpkg)
dpkg-query -Wf '${Installed-Size}\t${Package}\n' | sort -rn | head -20

# dnf
rpm -qa --queryformat '%{SIZE}\t%{NAME}\n' | sort -rn | head -20
```

## Очистка кэша

```bash
# pacman — хранит все скачанные пакеты
du -sh /var/cache/pacman/pkg/
sudo paccache -r                   # оставить 3 последних версии
sudo paccache -rk1                 # оставить только 1

# apt
du -sh /var/cache/apt/archives/
sudo apt clean                     # удалить весь кэш
sudo apt autoclean                 # удалить старые версии

# dnf
sudo dnf clean all
```

## Pinning версий (apt)

Зафиксировать версию пакета, чтобы `apt upgrade` не обновлял:

```bash
# Заблокировать обновление
sudo apt-mark hold package-name

# Разблокировать
sudo apt-mark unhold package-name

# Список заблокированных
apt-mark showhold
```

Для pacman — добавить в `/etc/pacman.conf`:
```ini
IgnorePkg = package-name
```

## Установка из исходников

```bash
# Установить инструменты сборки
# Ubuntu
sudo apt install build-essential

# Arch
sudo pacman -S base-devel

# Типичный процесс
tar xzf package-1.0.tar.gz
cd package-1.0
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
```

> **Совет:** Устанавливайте из исходников в `/usr/local/`, чтобы не конфликтовать с пакетным менеджером.

## Откат версии

```bash
# pacman — из кэша
sudo pacman -U /var/cache/pacman/pkg/package-oldversion.pkg.tar.zst

# apt — указать конкретную версию
apt list -a package-name           # доступные версии
sudo apt install package-name=1.2.3-1

# dnf
sudo dnf downgrade package-name
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Broken dependencies | apt: `sudo apt -f install`. pacman: `pacman -Syu` |
| Конфликт файлов | pacman: `--overwrite`. apt: `dpkg --force-overwrite` |
| `make install` мусорит в систему | Использовать `--prefix=/usr/local` или `checkinstall` |
| Пакет из PPA конфликтует | `ppa-purge` или вручную откатить |