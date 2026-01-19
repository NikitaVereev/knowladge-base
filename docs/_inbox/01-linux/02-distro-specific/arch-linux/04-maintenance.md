# Arch Maintenance

## Overview

Обслуживание Arch Linux система. Что делать каждый день, неделю, месяц.

**Что вы узнаете:**
- Регулярные обновления и их важность
- Очистка системы
- Мониторинг диска и памяти
- Оптимизация boot времени
- Резервные копии

## Prerequisites

- Установленный Arch Linux
- Знакомство с pacman и yay
- Базовое понимание systemd

## Ежедневное

### Обновление системы

```bash
sudo pacman -Syu               # КАЖДЫЙ ДЕНЬ!
yay                            # также обновить AUR пакеты
```

**Важно:** Arch — rolling release, обновления выходят постоянно.

### Мониторинг диска

```bash
df -h                          # свободное место на дисках
du -sh ~/*                     # что занимает место в home
du -sh /var/cache/pacman/pkg   # размер кэша pacman
```

## Еженедельно

### Очистка кэша

```bash
sudo pacman -Sc                # удалить кэш старых версий (безопасно)
sudo pacman -Qu                # какие пакеты можно обновить
```

### Orphaned пакеты

```bash
pacman -Qdtq                   # найти неиспользуемые зависимости
sudo pacman -Rs $(pacman -Qdtq) # удалить их
```

### Логи и tmp

```bash
sudo journalctl --vacuum=100M  # ограничить размер журнала
sudo rm -rf /tmp/*             # очистить tmp (осторожно!)
```

## Ежемесячно

### Полная очистка

```bash
# Кэш
sudo pacman -Scc               # удалить весь кэш pacman

# Журналы
sudo journalctl --vacuum=50M

# Временные файлы
sudo find /tmp -atime +7 -delete # удалить старые tmp файлы
```

### Оптимизация boot

```bash
systemd-analyze                # анализ boot времени
systemd-analyze blame          # что замедляет загрузку
systemd-analyze critical-chain # цепь зависимостей

# Отключите ненужные сервисы
sudo systemctl disable service # отключить при загрузке
sudo systemctl list-unit-files --state=enabled # какие включены
```

## Мониторинг системы

```bash
top                            # real-time мониторинг
htop                           # красивый top
watch df -h                    # следить за диском

# Температура CPU (если установлены sensors)
sensors
sudo pacman -S lm_sensors
sensors-detect
sensors
```

## Обновления и проблемы

### После крупного обновления

```bash
# Если что-то сломалось
sudo pacman -Syu --force
sudo mkinitcpio -P             # пересоздать initramfs

# Проверить целостность системы
pacman -Qk                     # проверить что все файлы на месте
```

### Если booting проблемы

```bash
# Восстановление GRUB
sudo grub-install --target=x86_64-efi --efi-directory=/efi
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## Резервные копии

```bash
# Список установленных пакетов
pacman -Qqe > pkglist.txt      # явно установленные
pacman -Qmq > aurpkgs.txt      # AUR пакеты

# Восстановление после чистой установки
sudo pacman -S - < pkglist.txt
yay -S - < aurpkgs.txt
```

## Key Takeaways

- **Arch требует регулярных обновлений** — не игнорируйте
- **Очистка экономит место** — `sudo pacman -Sc` еженедельно
- **Orphaned пакеты** — `sudo pacman -Rs $(pacman -Qdtq)` ежемесячно
- **Boot оптимизация** — `systemd-analyze` показывает проблемы
- **Резервные копии** — сохраняйте список пакетов

## Related

- [[02-pacman-guide|Pacman]] — управление пакетами
- [[docs/_inbox/01-linux/02-distro-specific/arch-linux/05-troubleshooting|Troubleshooting]] — решение проблем
- [[docs/_inbox/01-linux/02-distro-specific/README|Arch Index]] — индекс

## See Also

- [System maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [Systemd](https://wiki.archlinux.org/title/Systemd)
