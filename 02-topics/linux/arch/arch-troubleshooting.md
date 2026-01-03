---
created: 2026-01-03
tags: [arch-linux, troubleshooting, system-administration, reference]
type: reference
---

# Arch Linux Troubleshooting - решение проблем и восстановление

## Основная идея

Arch Linux требует умения диагностировать и решать проблемы.

**Методология troubleshooting:**
1. **Определить** - что не работает, когда началось
2. **Диагностировать** - посмотреть логи, проверить ошибки
3. **Изолировать** - понять в чём именно проблема
4. **Решить** - применить исправление
5. **Проверить** - убедиться что работает

---

## ЧАСТЬ 1: Диагностика - инструменты и методы

### Первая помощь (стандартная процедура)

```bash
# ШАГ 1: Посмотреть статус системы
systemctl status

# Показывает общее состояние системы

# ШАГ 2: Посмотреть критические ошибки
journalctl -p 3 -n 50 --no-pager

# -p 3 = только ошибки и выше
# -n 50 = последние 50 строк
# --no-pager = без pager'а (прямо в консоль)

# ШАГ 3: Если проблема с сервисом
systemctl --failed

# ШАГ 4: Посмотреть логи сервиса
sudo journalctl -u problem-service -n 100 --no-pager

# ШАГ 5: Если система не загружается
# Загрузиться с live USB
# Или загрузиться в другой kernel (GRUB)
```

### Логирование - главное оружие

```bash
# Real-time логи всей системы
journalctl -f

# Real-time логи конкретного сервиса
sudo journalctl -u service-name -f

# Логи за последний час
journalctl --since "1 hour ago"

# Логи между двумя временами
journalctl --since "2026-01-03 10:00:00" --until "2026-01-03 11:00:00"

# Только ошибки за последний день
journalctl -p err --since "1 day ago"

# Экспортировать в JSON для анализа
journalctl -o json | jq . > logs.json

# Поиск по тексту
journalctl | grep "error pattern"

# Статистика: сколько ошибок за месяц?
journalctl --since "1 month ago" -p err | wc -l
```

### Проверка аппаратного обеспечения

```bash
# ПРОЦЕССОР
lscpu                    # Информация о CPU
cat /proc/cpuinfo        #详альная информация CPU

# ПАМЯТЬ
free -h                  # Использование памяти
cat /proc/meminfo        # Подробная информация

# ДИСК
lsblk                    # Список дисков и партиций
df -h                    # Использование дискового пространства
du -sh /*                # Размер папок в root

# ТЕМПЕРАТУРА
sensors                  # Температура CPU (требует lm_sensors)
sudo pacman -S lm_sensors
sudo sensors-detect      # Первый раз запустить для конфигурации

# ЗДОРОВЬЕ ДИСКА (SMART)
sudo smartctl -a /dev/nvme0n1    # SSD информация
sudo smartctl -H /dev/nvme0n1    # Status здоровья

# ПАРТИЦИИ И ФАЙЛОВАЯ СИСТЕМА
sudo fsck /dev/partition        # Проверить файловую систему (требует unmount)
mount | grep "on / "            # Какая FS используется для root

# ПРОЦЕССЫ
ps aux                   # Все процессы
ps aux --sort=-%cpu | head    # Top процессы по CPU
ps aux --sort=-%mem | head    # Top процессы по памяти

# СЕТЬ
ip a                     # IP адреса
ip link                  # Сетевые интерфейсы
netstat -tulpn           # Открытые порты (требует net-tools)
ss -tulpn                # Открытые порты (современный аналог)
```

### Дисковый I/O мониторинг

```bash
# Установка инструментов
sudo pacman -S iotop sysstat

# Real-time I/O мониторинг
sudo iotop

# Или через systemd
systemd-cgtop

# Или статистика по процессам
pidstat -d 1             # Дисковый I/O по процессам каждую секунду
```

---

## ЧАСТЬ 2: Проблемы при загрузке

### Система не загружается (чёрный экран)

```bash
# ЧТО ДЕЛАТЬ:

# 1. Попробовать загрузиться в другом kernel'е
# На экране GRUB: выбрать Linux (другую версию) если есть

# 2. Если зависает на конкретном сообщении:
# На экране GRUB: нажать 'e' (edit)
# Найти строку с kernel параметрами
# Добавить в конец: nomodeset (отключить графику)
# Ctrl+X для загрузки

# 3. Если совсем не загружается - live USB
# Загрузиться с live USB (Arch ISO)
# Смонтировать систему:
sudo mount /dev/nvme0n1p2 /mnt
sudo arch-chroot /mnt

# 4. Проверить что сломалось:
journalctl -p err -n 100

# 5. Переустановить kernel
pacman -S linux

# 6. Выход и перезагрузка
exit
sudo reboot
```

### Kernel panic / Oops (система падает сразу после загрузки)

```bash
# Признаки:
# - Экран с текстом "Kernel panic"
# - Стек вызовов (backtrace)
# - Система перезагружается автоматически

# РЕШЕНИЕ:

# 1. Загрузиться в live USB
# 2. Mount систему
sudo mount /dev/nvme0n1p2 /mnt
sudo arch-chroot /mnt

# 3. Посмотреть последние логи (если доступны)
journalctl -p crit -n 50

# 4. Проверить целостность пакетов
pacman -Qk | grep -v OK

# 5. Если много пакетов - переустановить их
pacman -R problem-package
pacman -S problem-package

# 6. Переустановить kernel и initramfs
pacman -S linux mkinitcpio
mkinitcpio -P

# 7. Если ничего не помогает - откатить обновления
# Требует снимков (btrfs)
```

### GRUB не загружается или не видит систему

```bash
# РЕШЕНИЕ:

# 1. Загрузиться с live USB
sudo mount /dev/nvme0n1p2 /mnt
sudo mount /dev/nvme0n1p1 /mnt/boot
sudo arch-chroot /mnt

# 2. Переустановить GRUB
pacman -S grub efibootmgr

# 3. Переустановить GRUB в EFI
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

# 4. Пересоздать конфиг
grub-mkconfig -o /boot/grub/grub.cfg

# 5. Выход и перезагрузка
exit
sudo umount -R /mnt
sudo reboot
```

---

## ЧАСТЬ 3: Проблемы с пакетами

### Ошибка при установке: "target not found"

```bash
# Проблема:
sudo pacman -S nonexistent-package
# error: target not found: nonexistent-package

# РЕШЕНИЕ:

# 1. Проверить написание
pacman -Ss "search term"

# 2. Пакет может быть в AUR
yay -Ss "search term"

# 3. Пакет может быть удален
# Найти альтернативу:
pacman -Ss alternative

# 4. Пакет может быть в другом репо
# Включить multilib если нужны 32-bit пакеты:
sudo nano /etc/pacman.conf
# Раскомментировать:
# [multilib]
# Include = /etc/pacman.d/mirrorlist
```

### Ошибка зависимостей: "could not satisfy dependencies"

```bash
# Проблема:
sudo pacman -S package-name
# error: target not found: some-dependency
# error: failed to prepare transaction

# РЕШЕНИЕ:

# 1. Обновить систему (часто это помогает)
sudo pacman -Syu

# 2. Посмотреть какие зависимости требуются
pacman -Si package-name | grep -E "Depends|MakeDepends"

# 3. Установить зависимость вручную
sudo pacman -S dependency-name

# 4. Или установить целой группой
sudo pacman -S base-devel

# 5. Если зависимость несовместима с другим пакетом:
# Одну из них нужно удалить
sudo pacman -R conflicting-package
sudo pacman -S package-name

# 6. Если конфликт неразрешим:
# Использовать AUR версию или альтернативный пакет
```

### Пакет установлен но файлы не работают

```bash
# Проблема:
sudo pacman -S gcc
gcc --version
# gcc: command not found

# РЕШЕНИЕ:

# 1. Проверить что пакет установлен
pacman -Qi gcc

# 2. Посмотреть где находятся файлы
pacman -Ql gcc | grep bin

# Вывод: /usr/bin/gcc

# 3. Проверить что файл существует
ls -la /usr/bin/gcc

# 4. Проверить PATH
echo $PATH

# Если /usr/bin не в PATH - добавить в ~/.bashrc

# 5. Если файл удален
pacman -Qs gcc

# 6. Переустановить пакет
sudo pacman -S --force gcc
```

### Пакет требует удаления другого пакета

```bash
# Проблема:
sudo pacman -S new-package
# error: new-package and old-package are in conflict

# РЕШЕНИЕ:

# Способ 1: Принудительная установка (ОПАСНО!)
sudo pacman -S --force new-package
# Может поломать систему!

# Способ 2: Удалить конфликтующий пакет
sudo pacman -R old-package
sudo pacman -S new-package

# Способ 3: Найти альтернативу
# new-package может быть несовместим
pacman -Ss "alternative new-package"

# Способ 4: Использовать другую версию
pacman -Qs new-package
# Может быть -bin версия которая не конфликтует
sudo pacman -S new-package-bin

# Способ 5: Посмотреть на Arch Forums
# https://bbs.archlinux.org
# Часто конфликты уже обсуждались
```

### .pacnew и .pacsave файлы конфигов

```bash
# Проблема: При обновлении пакета создаются новые конфиги

# ДИАГНОСТИКА:

# Найти все .pacnew файлы
sudo find /etc -name "*.pacnew"

# Найти все .pacsave файлы
sudo find /etc -name "*.pacsave"

# РЕШЕНИЕ:

# Для каждого файла:

# 1. Посмотреть разницу
sudo diff /etc/package.conf /etc/package.conf.pacnew

# 2. Если новый конфиг лучше:
sudo mv /etc/package.conf.pacnew /etc/package.conf

# 3. Если нужен старый:
sudo rm /etc/package.conf.pacnew

# 4. Если оба нужны (merge):
sudo meld /etc/package.conf /etc/package.conf.pacnew
# Или вручную отредактировать

# АВТОМАТИЗАЦИЯ:

# Утилита для управления .pacnew файлами
sudo pacman -S pacdiff

# Показать все .pacnew файлы
sudo pacdiff

# Отредактировать с помощью vimdiff
sudo pacdiff -u
```

---

## ЧАСТЬ 4: Проблемы с производительностью

### Система очень медленная

```bash
# ДИАГНОСТИКА:

# 1. Процессы заедающие CPU
top
# или
htop

# 2. Процессы заедающие память
free -h
ps aux --sort=-%mem | head

# 3. Дисковый I/O
sudo iotop

# 4. Сетевой трафик
sudo nethogs

# РЕШЕНИЕ:

# Если процесс зависает:
kill -9 PID

# Если система вся медленная:
# - Перезагрузиться
# - Проверить диск (df -h)
# - Посмотреть логи ошибок (journalctl -p err)
# - Отключить ненужные сервисы (systemctl list-units)
```

### Диск медленный (большой latency)

```bash
# ДИАГНОСТИКА:

# Посмотреть stats диска
iostat -x 1

# Где:
# %iowait = процент времени в ожидании I/O
# await = среднее время операции
# svctm = время обработки

# Высокий iowait (>50%) = проблема с диском

# РЕШЕНИЕ:

# 1. Проверить здоровье диска
sudo smartctl -a /dev/nvme0n1

# 2. Если SMART показывает ошибки = диск умирает
# Срочно backup!

# 3. Проверить файловую систему
sudo fsck /dev/partition

# 4. Включить TRIM для SSD (помогает производительности)
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# 5. Если дисковый кэш поломан:
# Перезагрузиться
# Или отключить интенсивные процессы
```

### Утечка памяти (медленно растет использование)

```bash
# ДИАГНОСТИКА:

# Посмотреть память по времени
watch -n 1 free -h

# Найти процесс который растет
ps aux --sort=-%mem

# РЕШЕНИЕ:

# 1. Перезагрузить систему
sudo reboot

# 2. Если проблема повторяется:
# Определить какой сервис/приложение
sudo journalctl -f | grep -i "memory\|alloc"

# 3. Увеличить swap (временное решение)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 4. Постоянно добавить swap в /etc/fstab
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# 5. Переустановить проблемный пакет
sudo pacman -R problem-package
sudo pacman -S problem-package
```

### OOM Killer (система убивает процессы из-за нехватки памяти)

```bash
# Признаки:
# - Процессы неожиданно падают
# - journalctl показывает "Killed" сообщения
# - Нет ошибок в логах приложения

# РЕШЕНИЕ:

# 1. Посмотреть OOM сообщения
journalctl | grep -i "killed\|oom"

# 2. Увеличить оперативную память (железо)
# Или:

# 3. Увеличить swap
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 4. Уменьшить использование памяти
# Закрыть ненужные приложения
# Уменьшить размер кэша:
echo "vm.swappiness=50" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 5. Если конкретное приложение зависает:
# Ограничить его память через systemd
sudo systemctl edit application.service
# Добавить:
# [Service]
# MemoryMax=512M

sudo systemctl daemon-reload
sudo systemctl restart application.service
```

---

## ЧАСТЬ 5: Проблемы с сетью

### Нет интернета после загрузки

```bash
# ДИАГНОСТИКА:

# 1. Проверить интерфейсы
ip link
ip a

# 2. Пинг локального шлюза
ping 192.168.1.1

# 3. Пинг DNS сервера
ping 8.8.8.8

# 4. Посмотреть маршруты
ip route

# 5. Посмотреть DNS
cat /etc/resolv.conf

# РЕШЕНИЕ:

# Если интерфейс down:
sudo ip link set eth0 up

# Если нет IP адреса (DHCP):
sudo dhclient eth0
# или
sudo systemctl restart dhcpcd

# Если DHCP не работает - статический IP:
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip route add default via 192.168.1.1

# Если DNS не работает:
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

# Если wifi не подключается:
sudo systemctl restart iwd
iwctl
# device list
# station wlan0 scan
# station wlan0 get-networks
# station wlan0 connect SSID
# quit
```

### DNS не работает (медленный интернет)

```bash
# ПРИЗНАКИ:
# - Пинг по IP работает
# - Пинг по имени не работает (google.com)

# ДИАГНОСТИКА:

# Тест DNS
nslookup google.com
dig google.com

# Посмотреть текущие DNS серверы
cat /etc/resolv.conf

# РЕШЕНИЕ:

# 1. Изменить DNS вручную
sudo nano /etc/resolv.conf
# Заменить на:
# nameserver 1.1.1.1  (Cloudflare)
# nameserver 8.8.8.8  (Google)

# 2. Или через systemd-resolved (лучше)
sudo systemctl enable --now systemd-resolved
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf

# 3. Очистить DNS кэш
sudo resolvectl flush-caches

# 4. Перезагрузить сетевой сервис
sudo systemctl restart systemd-networkd
```

### Wifi не подключается или часто отваливается

```bash
# ДИАГНОСТИКА:

# Список доступных сетей
sudo iwctl
# > device list
# > station wlan0 get-networks

# Сигнал
sudo iwctl
# > station wlan0 get-diagnostics

# РЕШЕНИЕ:

# 1. Переподключиться
sudo iwctl
# > station wlan0 disconnect
# > station wlan0 connect SSID
# > station wlan0 show

# 2. Если часто отваливается - драйвер проблема:
lspci | grep -i wireless

# Найти драйвер для вашей карты:
pacman -Ss broadcom
# или для Intel:
pacman -Ss iwlwifi

# 3. Если мощность сигнала слаба:
# Переместиться ближе к роутеру
# Или использовать кабель

# 4. Перезагрузить wifi модуль
sudo modprobe -r iwlwifi
sudo modprobe iwlwifi
```

---

## ЧАСТЬ 6: Проблемы с услугами (systemd)

### Сервис не запускается

```bash
# ДИАГНОСТИКА:

# 1. Посмотреть статус
sudo systemctl status service-name

# 2. Посмотреть последние логи
sudo journalctl -u service-name -n 50

# 3. Попробовать запустить вручную
/usr/bin/service-executable

# РЕШЕНИЕ:

# Если "No such file or directory":
# - Пакет не установлен: sudo pacman -S package-name
# - Неправильный путь в .service файле

# Если "Permission denied":
# - Файл не имеет прав на исполнение: chmod +x
# - Запускается не под правильным пользователем: User= в .service

# Если "Address already in use":
# - Порт занят другим приложением
# - sudo netstat -tulpn | grep :8080

# Если "Connection refused":
# - Зависимость (БД, другой сервис) не запущена
# - Проверить After= и Requires= в .service
```

### Сервис постоянно падает и перезапускается

```bash
# ПРИЗНАКИ:
# systemctl status показывает: "Restart: in 5s"

# ДИАГНОСТИКА:

# Посмотреть логи
sudo journalctl -u service-name -f

# Искать ошибку в логах

# РЕШЕНИЕ:

# 1. Временно отключить auto-restart для отладки
sudo systemctl edit service-name
# [Service]
# Restart=no

sudo systemctl daemon-reload
sudo systemctl restart service-name

# Теперь сервис не будет перезапускаться
# Можно спокойно смотреть логи

# 2. После исправления проблемы:
# Вернуть Restart=on-failure

# 3. Общие причины:
# - Неправильный конфиг приложения
# - Отсутствие зависимости (файл, БД, сокет)
# - Недостаточно ресурсов (памяти, файловых дескрипторов)
# - Приложение требует интерактивного ввода
```

### Сервис зависает при остановке (timeout)

```bash
# ПРИЗНАКИ:
# systemctl stop занимает долго
# или
# Ошибка: "operation timed out"

# РЕШЕНИЕ:

# 1. Проверить TimeoutStopSec в .service
sudo systemctl cat service-name | grep TimeoutStop

# 2. Если нужно больше времени:
sudo systemctl edit service-name
# [Service]
# TimeoutStopSec=60

# 3. Если приложение не отвечает на SIGTERM:
# [Service]
# ExecStop=/bin/kill -9 $MAINPID

# 4. После изменения:
sudo systemctl daemon-reload
sudo systemctl restart service-name
```

---

## ЧАСТЬ 7: Проблемы с файловой системой

### Диск полный (нет места)

```bash
# ДИАГНОСТИКА:

# 1. Посмотреть какой раздел полный
df -h

# /dev/nvme0n1p2  100G  100G    0B 100%  /

# 2. Найти большие файлы/папки
du -sh /* | sort -h | tail

# 3. Если /var полная:
# Логи занимают место
du -sh /var/*

# РЕШЕНИЕ:

# 1. Очистить логи (БЕЗОПАСНО)
sudo journalctl --vacuum-time=7d
# или
sudo journalctl --vacuum-size=500M

# 2. Очистить кэш pacman (БЕЗОПАСНО)
sudo pacman -Sc

# 3. Очистить кэш yay (БЕЗОПАСНО)
yay -Sc --aur

# 4. Удалить временные файлы
sudo rm -rf /tmp/*
rm -rf ~/.cache/*

# 5. Если ничего не помогает - find большие файлы
sudo find / -xdev -type f -size +100M 2>/dev/null | head

# 6. Если /boot полная:
# Удалить старые kernel'ы (см. arch-maintenance)

# 7. Если критично - resize раздела (требует знаний)
```

### Файловая система повреждена (read-only)

```bash
# ПРИЗНАКИ:
# "Read-only file system"
# Не можно писать файлы

# ДИАГНОСТИКА:

# Посмотреть статус
mount | grep "on / "

# РЕШЕНИЕ:

# 1. Быстрое решение - перемонтировать RW
sudo mount -o remount,rw /

# 2. Если это не помогает - перезагрузиться
sudo reboot

# 3. Если проблема повторяется:
# Проверить файловую систему (требует live USB)
sudo umount /mount-point
sudo fsck /dev/partition
# Нажать 'y' для исправления ошибок

# 4. Если диск совсем сломан:
# Требует восстановления данных или переустановки

# ПРОФИЛАКТИКА:

# Проверять регулярно:
sudo fsck.ext4 -n /dev/partition  # -n = только проверка, не исправление
```

---

## ЧАСТЬ 8: Восстановление из backup'а

### Восстановить систему из снимка (если использовали btrfs)

```bash
# ТРЕБОВАНИЯ:
# - Файловая система btrfs
# - Включены снимки (snapshots)

# ВОССТАНОВЛЕНИЕ:

# 1. Загрузиться в live USB

# 2. Смонтировать root раздел
sudo mount /dev/nvme0n1p2 /mnt

# 3. Посмотреть доступные снимки
sudo btrfs subvolume list /mnt

# 4. Восстановить из снимка
sudo btrfs subvolume delete /mnt/@
sudo btrfs subvolume snapshot /mnt/@snapshot-name /mnt/@

# 5. Перезагрузиться
sudo reboot
```

### Восстановить файлы из backup'а

```bash
# Если файлы удалены случайно

# ДИАГНОСТИКА:

# 1. Проверить trash
ls -la ~/.local/share/Trash/files/

# 2. Если файлы там - восстановить из trash

# ЕСЛИ BACKUP ЕСТЬ:

# 1. Смонтировать backup раздел/диск
sudo mount /path/to/backup /mnt/backup

# 2. Скопировать файлы обратно
sudo cp -r /mnt/backup/home/user/documents /home/user/

# 3. Проверить что скопировалось
ls -la /home/user/documents

# 4. Исправить владельца если нужно
sudo chown -R user:user /home/user/documents
```

---

## ЧАСТЬ 9: Шпаргалка (быстрый справочник)

### Инструменты диагностики

```bash
# СИСТЕМНЫЕ ЛОГИ
journalctl -p 3 -n 50           # Критические ошибки
journalctl -p err -b            # Ошибки с загрузки
systemctl status                # Статус системы
systemctl --failed              # Упавшие сервисы

# АППАРАТНОЕ ОБЕСПЕЧЕНИЕ
lscpu                           # Информация CPU
free -h                         # Память
df -h                           # Диски
sensors                         # Температура
sudo smartctl -a /dev/nvme0n1   # Здоровье диска
ps aux --sort=-%cpu            # Процессы по CPU

# ДИСКОВЫЙ I/O
sudo iotop                      # I/O мониторинг
iostat -x 1                     # I/O статистика
du -sh /*                       # Размер папок

# СЕТЕВЫЕ ПРОВЕРКИ
ping 8.8.8.8                    # Тест интернета
nslookup google.com             # Тест DNS
ip a                            # IP адреса
sudo netstat -tulpn             # Открытые порты

# СЕРВИСЫ
systemctl status service-name           # Статус сервиса
sudo journalctl -u service -f          # Логи сервиса
sudo systemctl restart service-name    # Перезагрузить
```

### Быстрые исправления

```bash
# СИСТЕМА НЕ ЗАГРУЖАЕТСЯ
# -> Загрузиться с live USB
# -> arch-chroot /mnt
# -> pacman -S linux

# СЕРВИС НЕ ЗАПУСКАЕТСЯ
# -> sudo journalctl -u service -n 50
# -> Посмотреть ошибку и исправить

# СИСТЕМА МЕДЛЕННАЯ
# -> top (посмотреть CPU)
# -> free -h (посмотреть память)
# -> sudo iotop (посмотреть I/O)

# ДИСК ПОЛНЫЙ
# -> sudo journalctl --vacuum-time=7d
# -> sudo pacman -Sc
# -> yay -Sc --aur

# НЕТ ИНТЕРНЕТА
# -> ip link (проверить интерфейсы)
# -> sudo dhclient eth0 (DHCP)
# -> ping 8.8.8.8 (тест сети)
# -> nslookup google.com (тест DNS)

# СЕРВИС ПОСТОЯННО ПАДАЕТ
# -> sudo journalctl -u service -f
# -> Посмотреть причину в логах
# -> Исправить конфиг или переустановить пакет
```

---

## ЧАСТЬ 10: Когда нужна переустановка

### Признаки что нужна переустановка

```
□ Система совсем не загружается (даже с live USB)
□ Файловая система совсем повреждена (fsck не помогает)
□ Диск умирает (SMART показывает ошибки)
□ Множество проблем которые не решаются (не рекомендуется)
```

### Минимально безболезненная переустановка

```bash
# Если нужно переустановить сохранив /home

# 1. Загрузиться с live USB

# 2. Сделать backup /home если его нет
sudo mount /dev/nvme0n1p3 /mnt/home
sudo cp -r /mnt/home /tmp/home-backup

# 3. Форматировать только / раздел
sudo mkfs.ext4 /dev/nvme0n1p2

# 4. Установить Arch как обычно
# Но когда выбираем что монтировать:
# /dev/nvme0n1p2 -> /
# /dev/nvme0n1p3 -> /home (НЕ форматировать!)
# /dev/nvme0n1p1 -> /boot

# 5. После установки:
# Все конфиги и файлы будут на месте
# Просто переустановите пакеты
```

---

## Связанные заметки

- [[arch-maintenance]] - регулярное обслуживание Arch
- [[systemd-basics]] - systemd система инициализации
- [[pacman-basic-commands]] - менеджер пакетов
- [[aur-yay-commands]] - AUR и yay
- [[linux-backup-strategy]] - стратегия backup'ов

## Источники

- Arch Wiki: Troubleshooting
- Arch Linux Forum: https://bbs.archlinux.org
- Linux man pages: man journalctl, man fsck, man pacman
- ArchLinux.org документация

---

Создано: 2026-01-03 19:32
