---
title: "Обслуживание Arch Linux"
type: how-to
tags: [linux, arch, maintenance, update, cleanup, snapshot, pacnew]
sources:
  original: "_inbox/01-linux/02-distro-specific/arch-linux/04-maintenance.md"
related:
  - "[[linux/how-to/arch/pacman-and-aur]]"
  - "[[linux/how-to/arch/troubleshooting]]"
---

# Обслуживание Arch Linux

> **TL;DR:** Обновляйте минимум раз в неделю. Читайте archlinux.org/news перед обновлением.
> Чистите кэш pacman, удаляйте orphans, проверяйте .pacnew файлы.

## Регулярное обновление

```bash
# Перед обновлением — проверить новости!
# https://archlinux.org/news/

# Обновить всё
sudo pacman -Syu

# С AUR
yay -Syu
```

> **Правило:** Обновляйте не реже раза в неделю. Длительные перерывы (>1 месяц) увеличивают риск проблем.

## Очистка системы

```bash
# Кэш pacman (оставить 3 последних версии)
sudo paccache -r
# Оставить только 1 версию
sudo paccache -rk1

# Удалить orphan-пакеты
sudo pacman -Rns $(pacman -Qdtq)

# Очистить пользовательский кэш
rm -rf ~/.cache/*

# Журнал systemd (оставить за 2 недели)
sudo journalctl --vacuum-time=2weeks

# Проверить размер кэша
du -sh /var/cache/pacman/pkg/
du -sh ~/.cache/
```

## .pacnew и .pacsave файлы

При обновлении pacman может создать `.pacnew` (новая версия конфига) или `.pacsave` (ваш конфиг при удалении пакета).

```bash
# Найти все .pacnew
sudo find /etc -name "*.pacnew" 2>/dev/null

# Сравнить и объединить
sudo diff /etc/pacman.conf /etc/pacman.conf.pacnew
sudo vim -d /etc/pacman.conf /etc/pacman.conf.pacnew

# После объединения — удалить
sudo rm /etc/pacman.conf.pacnew

# Автоматически (pacdiff)
sudo pacdiff
```

## Бэкап списка пакетов

```bash
# Сохранить список явно установленных пакетов
pacman -Qqe > pkglist.txt
pacman -Qqem > aurlist.txt           # только AUR

# Восстановить на другой системе
sudo pacman -S --needed - < pkglist.txt
yay -S --needed - < aurlist.txt
```

## Мониторинг здоровья

```bash
# Проверить ошибки systemd
systemctl --failed

# Проверить логи на ошибки
journalctl -p 3 -b                  # errors с последней загрузки

# Состояние дисков
df -h
sudo smartctl -a /dev/sda           # SMART (нужен smartmontools)
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Не обновлял месяц, всё сломалось | Читать archlinux.org/news, обновлять чаще |
| `.pacnew` накопились | `sudo pacdiff` — регулярно проверять |
| `/boot` заполнен старыми ядрами | Arch хранит только текущее. Проверить `ls /boot/` |
| Broken symlinks | `find / -xtype l 2>/dev/null` |