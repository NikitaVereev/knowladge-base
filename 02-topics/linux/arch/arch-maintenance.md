---
created: 2026-01-01
tags: [arch-linux, maintenance, system-administration, reference]
type: reference
---

# Arch Linux Maintenance - регулярные задачи и чеклист

## Основная идея

Arch Linux требует регулярного обслуживания для стабильной работы (rolling release).

**Три уровня maintenance:**
- **Еженедельно** - быстрые проверки (15 мин)
- **Ежемесячно** - полная очистка (30 мин)
- **Раз в квартал** - глубокая проверка (1-2 часа)

---

## ЧАСТЬ 1: Еженедельное обслуживание (ОБЯЗАТЕЛЬНО!)

### Обновление системы

```bash
# САМОЕ ВАЖНОЕ! Выполняйте каждую неделю минимум
sudo pacman -Syu

# Что это делает:
# 1. Обновляет список пакетов из репозиториев
# 2. Проверяет какие пакеты можно обновить
# 3. Скачивает и устанавливает обновления
# 4. Обновляет даже системные компоненты (kernel, systemd)
```

**Почему каждую неделю?**
- Arch Linux - rolling release (постоянные обновления)
- Security patche'ы важны
- Если не обновляться месяц - могут быть проблемы с зависимостями
- Breaking changes требуют скорой реакции

### Проверка ошибок сервисов

```bash
# Посмотреть какие сервисы упали
systemctl --failed

# Вывод:
#  UNIT                                   LOAD   ACTIVE SUB    DESCRIPTION
#  ● some-service.service              loaded failed failed  Some Service
#
# 1 loaded units listed.

# Если есть failed сервисы:
sudo systemctl status some-service      # Посмотреть ошибку
sudo journalctl -u some-service -n 50   # Посмотреть логи
sudo systemctl restart some-service     # Попробовать перезапустить
```

### Проверка логов (ошибки системы)

```bash
# Посмотреть критические ошибки в логах за эту загрузку
journalctl -p 3 -xb

# Параметры:
# -p 3 = только критические ошибки (приоритет 3: err)
# -x = расширенный формат (с объяснениями)
# -b = с текущей загрузки

# Приоритеты:
# 0=emerg, 1=alert, 2=crit, 3=err, 4=warning, 5=notice, 6=info, 7=debug

# Другие варианты:
journalctl -p err -b         # Ошибки и выше
journalctl -p warning -b     # Предупреждения и выше
journalctl --since "1 hour ago" -p err  # Ошибки за последний час
```

**Что ищем в логах:**
- `kernel panic` - критическая ошибка ядра
- `segmentation fault` - ошибка памяти приложения
- `ACPI errors` - проблемы с железом/BIOS
- `cannot open /dev/` - проблемы с устройствами
- Повторяющиеся ошибки сервисов

### Проверка диска

```bash
# Посмотреть место на диске
df -h

# Вывод:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/nvme0n1p2  100G   45G   55G  45%  /
# /dev/nvme0n1p1  512M  512M    0B 100%  /boot  ← ВНИМАНИЕ!

# Если /boot 100% - нужно очистить старые kernel'ы

# Посмотреть размер папок
du -sh ~/*          # Размер папок в home
du -sh /var/*       # Размер папок в /var
```

**Если /boot полный:**
```bash
# Посмотреть старые kernel'ы
ls -la /boot | grep vmlinuz

# Удалить старые через pacman (или вручную)
sudo pacman -R linux-headers-5.15  # старая версия

# Проверить что осталось место
df -h /boot
```

### Быстрая еженедельная процедура

```bash
#!/bin/bash
# Сохраните как ~/weekly-maintenance.sh

echo "=== Weekly Arch Maintenance ==="

echo "1. Updating system..."
sudo pacman -Syu --noconfirm

echo "2. Checking failed services..."
systemctl --failed

echo "3. Checking errors in logs..."
journalctl -p 3 -xb | tail -20

echo "4. Checking disk space..."
df -h / /boot

echo "5. Cleaning AUR dependencies..."
yay -Yc --noconfirm

echo "=== Done! ==="
```

**Использование:**
```bash
chmod +x ~/weekly-maintenance.sh
./weekly-maintenance.sh

# Или запускать еженедельно через cron:
# Добавить в crontab -e:
# 0 2 * * 0 /home/user/weekly-maintenance.sh >> /tmp/maintenance.log 2>&1
```

---

## ЧАСТЬ 2: Ежемесячное обслуживание

### Очистка кэша пакетов

```bash
# Удалить старые версии пакетов из кэша (БЕЗОПАСНО)
sudo pacman -Sc

# Что это делает:
# - Удаляет скачанные пакеты старых версий
# - Оставляет текущие версии (на случай переустановки)
# - Освобождает место на диске

# Сколько места займет?
du -sh /var/cache/pacman/pkg

# Пример:
# 3.2G /var/cache/pacman/pkg (до очистки)
# После pacman -Sc:
# 1.5G /var/cache/pacman/pkg (только текущие версии)
```

### Очистка AUR кэша

```bash
# Очистить кэш yay (папки пакетов которые собирались)
yay -Sc --aur

# Или вручную:
rm -rf ~/.cache/yay/*

# Сколько займет?
du -sh ~/.cache/yay

# Может быть несколько гигабайт если много AUR пакетов!
```

### Проверка системных логов

```bash
# Посмотреть ошибки за весь месяц
journalctl --since "1 month ago" -p err

# Найти повторяющиеся ошибки
journalctl -p err | sort | uniq -c | sort -rn | head

# Вывод может быть:
#      15 error: Could not load module xyz
#      12 kernel: Out of memory
#       8 systemd: Unit failed to start

# Это признак что нужно:
# - Переустановить пакет xyz
# - Увеличить swap (если out of memory)
# - Перезагрузиться
```

### Проверка температуры и здоровья железа

```bash
# Посмотреть температуру CPU
sensors

# Вывод:
# Core 0:       +45°C  (high = +80°C, crit = +100°C)
# Core 1:       +42°C
# Package id 0: +45°C

# Если выше 80°C - проверить вентилятор!

# Проверить здоровье SSD (SMART данные)
sudo smartctl -a /dev/nvme0n1

# Вывод:
# SMART Health Status: OK
# Temperature: 35°C
# Power On Hours: 2500

# Если "failing" или высокая температура - диск может скоро сломаться!
```

### Проверка обновления kernel'а

```bash
# Посмотреть текущий kernel
uname -r
# Вывод: 6.6.8-arch1-1

# Посмотреть доступный kernel
pacman -Qi linux | grep Version
# Вывод: Version : 6.7.1-arch1-1

# Если версия выше - обновить и перезагрузиться
sudo pacman -Syu
sudo reboot

# Проверить что используется новый kernel
uname -r
```

**Важно:** Всегда перезагружайтесь после обновления kernel'а!

### Проверка /var/log

```bash
# /var/log может занимать место, особенно логи systemd

# Посмотреть размер
du -sh /var/log

# Очистить логи старше 30 дней (БЕЗОПАСНО)
sudo journalctl --vacuum-time=30d

# Или по размеру (максимум 500MB журналов)
sudo journalctl --vacuum-size=500M

# Проверить результат
du -sh /var/log
```

### Полная ежемесячная процедура

```bash
#!/bin/bash
# Сохраните как ~/monthly-maintenance.sh

echo "=== Monthly Arch Maintenance ==="

echo "1. Updating system..."
sudo pacman -Syu --noconfirm

echo "2. Cleaning pacman cache..."
sudo pacman -Sc --noconfirm

echo "3. Cleaning yay cache..."
yay -Sc --aur --noconfirm

echo "4. Cleaning old journal logs..."
sudo journalctl --vacuum-time=30d

echo "5. Cleaning AUR orphans..."
yay -Yc --noconfirm

echo "6. Checking disk usage..."
echo "Root partition:"
df -h /
echo "Boot partition:"
df -h /boot

echo "7. Checking for errors..."
journalctl --since "1 month ago" -p err | wc -l

echo "8. Checking failed services..."
systemctl --failed

echo "9. Checking temperatures..."
sensors | grep "Core\|Package"

echo "=== Done! ==="
```

---

## ЧАСТЬ 3: Квартальная (раз в 3 месяца) проверка

### Глубокая проверка системы

```bash
# Проверить целостность всех установленных пакетов
sudo pacman -Qk

# Вывод:
# package-name: все файлы OK
# или
# package-name: файл не найден

# Если много файлов не найдено:
# Переустановить пакет:
sudo pacman -S --force package-name
```

### Проверка зависимостей

```bash
# Найти сломанные зависимости
pacman -Dd

# Вывод если есть проблемы:
# some-package: requires missing-dependency

# Решение:
sudo pacman -S missing-dependency
# или
sudo pacman -R some-package
```

### Обновление pacman-key'ей (архив подписей)

```bash
# Иногда ключи устаревают
sudo pacman-key --refresh-keys

# Это может занять время (5-10 мин) так как скачивает с сервера
```

### Проверка конфигов (/etc)

```bash
# Посмотреть файлы конфигов требующие обновления
sudo pacman -Qu | grep ".pacnew"

# При обновлении пакета обновляются конфиги
# Если конфиг не изменялся - заменяется автоматически
# Если менялся - создается .pacnew и .pacsave файл

# Найти все .pacnew файлы
sudo find /etc -name "*.pacnew"

# Посмотреть разницу
sudo diff /etc/package.conf /etc/package.conf.pacnew

# Если новый конфиг лучше:
sudo mv /etc/package.conf.pacnew /etc/package.conf

# Если нужно оставить старый:
sudo rm /etc/package.conf.pacnew
```

### Проверка AUR пакетов на обновления

```bash
# Посмотреть какие AUR пакеты устарели
yay -Qu

# Обновить только AUR (без официальных)
yay -Sua

# Или интерактивно выбрать какие обновлять
yay -Sua --ask 4
```

### Очистка home директории

```bash
# Найти большие файлы которые можно удалить
find ~/ -type f -size +100M -exec ls -lh {} \; | awk '{print $5, $9}'

# Очистить папку Downloads (старше 3 месяцев)
find ~/Downloads -type f -mtime +90 -delete

# Очистить temp папки
rm -rf ~/.cache/*
rm -rf ~/.local/share/Trash/*

# Но сохранить конфиги приложений:
# НЕ удаляйте ~/.config ~/.local/share/applications ~/.bashrc и т.д.
```

---

## ЧАСТЬ 4: После больших обновлений

### Если обновление сломало систему

```bash
# 1. Посмотреть что произошло
journalctl -p err -n 50

# 2. Попробовать rollback (если есть снимок)
# (Требует btrfs или снимков)

# 3. Если сервис не запускается:
sudo systemctl status problem-service
sudo systemctl restart problem-service

# 4. Если весь система не загружается - загрузиться в TTY:
# Нажать Ctrl+Alt+F2 (или F3-F6)
# Залогиниться
sudo systemctl emergency   # emergency shell
```

### Восстановление после kernel update

```bash
# Если система не загружается после обновления kernel'а:

# 1. Загрузиться с live USB
# 2. Mount root partition:
sudo mount /dev/nvme0n1p2 /mnt

# 3. Chroot в систему:
sudo arch-chroot /mnt

# 4. Переустановить kernel:
sudo pacman -S linux

# 5. Выход:
exit
sudo reboot
```

---

## ЧАСТЬ 5: Автоматизация maintenance (systemd timer)

### Создать systemd timer для weekly maintenance

`/etc/systemd/system/arch-maintenance.timer`:
```ini
[Unit]
Description=Weekly Arch Maintenance Timer

[Timer]
OnCalendar=Sun *-*-* 02:00:00    # Каждое воскресенье в 2:00
Persistent=yes

[Install]
WantedBy=timers.target
```

`/etc/systemd/system/arch-maintenance.service`:
```ini
[Unit]
Description=Weekly Arch Maintenance

[Service]
Type=oneshot
ExecStart=/usr/local/bin/arch-maintenance.sh
StandardOutput=journal
StandardError=journal
```

`/usr/local/bin/arch-maintenance.sh`:
```bash
#!/bin/bash
set -e

echo "Starting weekly Arch maintenance..."

# Обновление
pacman -Syu --noconfirm

# Очистка AUR
yay -Yc --noconfirm

# Очистка кэша pacman
pacman -Sc --noconfirm

# Очистка логов
journalctl --vacuum-time=30d

echo "Maintenance completed successfully"
```

**Установка:**
```bash
# 1. Создать файлы (копировать выше)
sudo nano /etc/systemd/system/arch-maintenance.timer
sudo nano /etc/systemd/system/arch-maintenance.service
sudo nano /usr/local/bin/arch-maintenance.sh

# 2. Выдать права на исполнение
sudo chmod +x /usr/local/bin/arch-maintenance.sh

# 3. Включить timer
sudo systemctl daemon-reload
sudo systemctl enable --now arch-maintenance.timer

# 4. Проверить
sudo systemctl list-timers arch-maintenance.timer
sudo journalctl -u arch-maintenance.service -f
```

---

## ЧАСТЬ 6: Мониторинг состояния системы

### Инструменты мониторинга

```bash
# Установка инструментов
sudo pacman -S htop iotop nethogs

# htop - мониторинг CPU и памяти
htop

# iotop - мониторинг дискового I/O
sudo iotop

# nethogs - мониторинг сетевого трафика
sudo nethogs

# Системные метрики в реальном времени
systemd-cgtop
```

### Проверка background процессов

```bash
# Посмотреть все процессы
ps aux | grep -v root | head -20

# Найти процессы заедающие CPU
ps aux --sort=-%cpu | head

# Найти процессы заедающие память
ps aux --sort=-%mem | head

# Убить процесс который зависает
kill -9 PID
```

---

## ЧАСТЬ 7: Чеклист maintenance

### Еженедельно (15 минут)

```
□ sudo pacman -Syu
□ systemctl --failed (проверить нет ли упавших сервисов)
□ journalctl -p 3 -xb (проверить критические ошибки)
□ df -h (проверить место на диске)
□ yay -Yc (очистить orphan пакеты)
```

### Ежемесячно (30 минут)

```
□ Выполнить еженедельный чеклист
□ sudo pacman -Sc (очистить кэш pacman)
□ yay -Sc --aur (очистить кэш yay)
□ sudo journalctl --vacuum-time=30d (очистить логи)
□ sensors (проверить температуру CPU)
□ sudo smartctl -a /dev/nvme0n1 (проверить SSD)
□ systemctl --failed (посмотреть failed services)
□ journalctl --since "1 month ago" -p err (ошибки за месяц)
```

### Раз в квартал (1-2 часа)

```
□ Выполнить ежемесячный чеклист
□ sudo pacman -Qk (проверить целостность пакетов)
□ pacman -Dd (проверить сломанные зависимости)
□ sudo pacman-key --refresh-keys (обновить ключи)
□ sudo find /etc -name "*.pacnew" (проверить .pacnew конфиги)
□ pacman -Qu (посмотреть устаревшие AUR пакеты)
□ yay -Sua (обновить AUR пакеты)
□ find ~/ -type f -size +100M (найти большие файлы)
□ systemd-analyze blame (посмотреть медленные юниты при загрузке)
```

---

## ЧАСТЬ 8: Решение типичных проблем

### Система очень медленная

```bash
# 1. Посмотреть что заедает CPU
top
# или
htop

# 2. Посмотреть дисковый I/O
sudo iotop

# 3. Посмотреть память
free -h

# 4. Если memory полная - увеличить swap
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Система не загружается

```bash
# 1. Загрузиться в GRUB menu
# 2. Выбрать старый kernel если возможно
# 3. Или загрузиться с live USB
# 4. Чинить систему (см. "Восстановление после kernel update")
```

### Часто падают сервисы

```bash
# 1. Посмотреть какой сервис падает
systemctl --failed

# 2. Посмотреть логи
sudo journalctl -u service-name -n 100

# 3. Обновить пакет
sudo pacman -S package-name

# 4. Или переустановить
sudo pacman -R package-name
sudo pacman -S package-name

# 5. Перезагрузить
sudo systemctl restart service-name
```

---

## ЧАСТЬ 9: Шпаргалка (быстрый справочник)

### Команды maintenance

```bash
# ОБНОВЛЕНИЕ
sudo pacman -Syu                 # Обновить всё
yay -Syu                         # Обновить + AUR

# ОЧИСТКА
sudo pacman -Sc                  # Очистить кэш pacman
yay -Sc --aur                    # Очистить кэш yay
yay -Yc                          # Очистить orphan пакеты
sudo journalctl --vacuum-time=30d

# ПРОВЕРКА
systemctl --failed               # Упавшие сервисы
journalctl -p 3 -xb              # Критические ошибки
df -h                            # Место на диске
pacman -Qk                       # Целостность пакетов
pacman -Dd                       # Сломанные зависимости

# ИНФОРМАЦИЯ
sensors                          # Температура CPU
uname -r                         # Текущий kernel
pacman -Qi kernel-package        # Информация о kernel'е
```

---

## Связанные заметки

- [[pacman-basic-commands]] - pacman менеджер пакетов
- [[aur-yay-commands]] - AUR и yay
- [[systemd-basics]] - systemd система инициализации
- [[arch-troubleshooting]] - решение проблем Arch
- [[linux-backup-strategy]] - стратегия backup'ов

## Источники

- Arch Wiki: Maintenance
- Arch Wiki: System maintenance
- Arch Linux official documentation

---
Создано: 2026-01-01 18:29
