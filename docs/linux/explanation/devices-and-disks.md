---
title: "Устройства, диски и файловые системы"
type: explanation
tags: [linux, devices, disks, udev, fdisk, lvm, ext4, fstab, mount, swap, partitions]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 4"
  docs: "https://man7.org/linux/man-pages/man8/lsblk.8.html"
related:
  - "[[linux/explanation/filesystem]]"
  - "[[linux/explanation/architecture]]"
  - "[[linux/how-to/manage-disks]]"
  - "[[linux/reference/cheatsheet]]"
---

# Устройства, диски и файловые системы

> **TL;DR:** В Linux устройства представлены файлами в `/dev/`. Блочные (`/dev/sda`) — диски с произвольным доступом. Символьные (`/dev/tty`) — потоковые. udev автоматически создаёт файлы устройств. Диски размечаются (MBR/GPT), на разделах создаются ФС (ext4, XFS), разделы монтируются в дерево каталогов. LVM добавляет гибкость поверх разделов.

## Зачем это знать

- Добавить диск к серверу — нужно разметить, создать ФС, смонтировать, прописать в fstab
- Понять вывод `lsblk`, `df`, `mount` — что куда смонтировано
- Проблемы с I/O — понимание уровней: устройство → драйвер → ФС → VFS → приложение
- Docker volumes, Kubernetes PV — всё это абстракции поверх блочных устройств и ФС

## Типы устройств

| Тип | Символ в `ls -l` | Доступ | Примеры |
|---|---|---|---|
| Блочное (block) | `b` | Произвольный (по блокам) | `/dev/sda`, `/dev/nvme0n1` — диски |
| Символьное (character) | `c` | Последовательный (поток) | `/dev/tty`, `/dev/null`, `/dev/urandom` |
| Именованный канал (pipe) | `p` | FIFO | Создаётся `mkfifo` |
| Сокет | `s` | Двусторонний | `/run/docker.sock` |

```bash
ls -la /dev/sda /dev/null
# brw-rw---- 1 root disk 8, 0 ... /dev/sda     ← b = block, 8,0 = major,minor
# crw-rw-rw- 1 root root 1, 3 ... /dev/null     ← c = character
```

### Номера major и minor

Каждое устройство идентифицируется парой чисел: **major** (тип драйвера) и **minor** (конкретное устройство или раздел).

```
/dev/sda   8, 0    ← major=8 (SCSI/SATA), minor=0 (диск целиком)
/dev/sda1  8, 1    ← major=8, minor=1 (первый раздел)
/dev/sda2  8, 2    ← major=8, minor=2 (второй раздел)
```

### Специальные устройства

| Устройство | Назначение |
|---|---|
| `/dev/null` | Чёрная дыра — поглощает всё записанное, при чтении даёт EOF |
| `/dev/zero` | Бесконечный поток нулевых байтов |
| `/dev/urandom` | Криптографически безопасные псевдослучайные данные |
| `/dev/full` | При записи всегда возвращает «диск полон» (ENOSPC) |
| `/dev/tty` | Текущий терминал процесса |
| `/dev/loop*` | Loopback — файл-образ как блочное устройство |

## /dev/, sysfs и udev

### devtmpfs и /dev/

`/dev/` — виртуальная ФС `devtmpfs`, живущая в RAM. Ядро создаёт в ней базовые узлы устройств при обнаружении оборудования.

### sysfs (/sys/)

`/sys/` — виртуальная ФС, отражающая представление ядра об устройствах, драйверах и их связях. Структура каталогов = топология оборудования.

```bash
# Информация о блочном устройстве
cat /sys/block/sda/size          # размер в секторах (512 байт)
cat /sys/block/sda/queue/scheduler   # планировщик I/O
ls /sys/block/sda/device/        # связь с драйвером
```

### udev

Демон `systemd-udevd` слушает события ядра (подключение/отключение устройств) и применяет правила из `/etc/udev/rules.d/` и `/usr/lib/udev/rules.d/`:

- Создаёт символические ссылки (`/dev/disk/by-uuid/...`, `/dev/disk/by-id/...`)
- Устанавливает права доступа и владельца
- Запускает скрипты при событиях

```bash
# Информация об устройстве
udevadm info --query=all --name=/dev/sda

# Мониторинг событий (вставьте USB-диск)
udevadm monitor

# Применить правила заново
udevadm trigger
```

## dd — копирование на уровне блоков

Копирует данные побайтово, минуя файловую систему.

```bash
# Записать ISO на USB
sudo dd if=ubuntu.iso of=/dev/sdX bs=4M status=progress

# Создать файл нулей (1 ГБ)
dd if=/dev/zero of=bigfile bs=1M count=1024

# Резервная копия MBR
sudo dd if=/dev/sda of=mbr-backup.bin bs=512 count=1
```

> **Осторожно:** `dd` не спрашивает подтверждения и может уничтожить данные. Трижды проверяйте `of=`.

## Разметка дисков

### MBR vs GPT

| | MBR | GPT |
|---|---|---|
| Макс. разделов | 4 основных (или 3+extended) | 128 |
| Макс. размер диска | 2 ТБ | 8 ЗБ |
| Таблица разделов | В первых 512 байтах | В начале и конце диска (дублирование) |
| Загрузка | BIOS | UEFI (или BIOS с BIOS Boot Partition) |
| Современность | Устарел | Стандарт |

### Инструменты разметки

```bash
# Интерактивные
sudo fdisk /dev/sda            # MBR и GPT (текстовый)
sudo cfdisk /dev/sda           # MBR и GPT (псевдографика)
sudo parted /dev/sda           # GPT (мощный, скриптуемый)

# Просмотр
lsblk                          # дерево блочных устройств
lsblk -f                       # с FS и UUID
sudo fdisk -l /dev/sda         # таблица разделов
sudo blkid                     # UUID всех разделов
```

## Файловые системы

### Типы ФС

| ФС | Особенности | Когда использовать |
|---|---|---|
| **ext4** | Журналирование, стабильность, широкая поддержка | Универсальный выбор по умолчанию |
| **XFS** | Высокая производительность на больших файлах, параллельный I/O | Серверы, БД, большие объёмы |
| **Btrfs** | Снапшоты, сжатие, RAID, copy-on-write | Настольные системы, сложные сценарии |
| **FAT32/VFAT** | Совместимость, без прав доступа | ESP (`/boot/efi`), USB-флешки |
| **tmpfs** | В оперативной памяти | `/tmp`, `/run` |
| **proc, sysfs** | Виртуальные (ядро) | `/proc`, `/sys` |

### Создание, проверка, настройка

```bash
# Создать ФС
sudo mkfs.ext4 /dev/sda2
sudo mkfs.xfs /dev/sda3
sudo mkfs.fat -F32 /dev/sda1          # для ESP

# Проверка и восстановление (раздел должен быть размонтирован!)
sudo fsck /dev/sda2
sudo fsck.ext4 -f /dev/sda2           # принудительная проверка

# Настройка ext4
sudo tune2fs -l /dev/sda2             # информация о ФС
sudo tune2fs -L "mydata" /dev/sda2    # метка
sudo tune2fs -c 30 /dev/sda2          # fsck каждые 30 монтирований
```

### Структура ext4

Каждый файл имеет **inode** (index node) — структуру данных с метаинформацией: владелец, права, размер, временные метки, указатели на блоки данных. Имя файла хранится в директории (маппинг «имя → номер inode»).

```bash
ls -i file              # показать номер inode
stat file               # полная информация inode
df -i                   # использование inode
```

Журнал (journal) записывает изменения до их применения — при сбое ФС может откатить незавершённые операции вместо полного fsck.

## Монтирование

Монтирование — подключение ФС раздела к точке в дереве каталогов.

```bash
# Ручное монтирование
sudo mount /dev/sda2 /mnt
sudo mount -t ext4 /dev/sda2 /mnt          # указать тип FS
sudo mount -o ro /dev/sda2 /mnt            # read-only
sudo mount UUID=xxx-yyy /mnt               # по UUID

# Размонтирование
sudo umount /mnt
sudo umount -l /mnt                        # lazy (если «busy»)
```

### /etc/fstab — автоматическое монтирование

Каждая строка — правило монтирования при загрузке:

```
# <устройство>                          <точка>   <тип>  <опции>       <dump> <fsck>
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /         ext4   errors=remount-ro 0    1
UUID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy /home     ext4   defaults          0    2
UUID=zzzzzzzz-zzzz                       /boot/efi vfat   umask=0077        0    1
UUID=ssssssss-ssss                       none      swap   sw                0    0
tmpfs                                    /tmp      tmpfs  defaults,size=2G  0    0
```

| Поле | Описание |
|---|---|
| Устройство | UUID (надёжно), `/dev/sda2`, LABEL=name |
| Точка монтирования | Путь в дереве каталогов |
| Тип FS | ext4, xfs, vfat, swap, tmpfs |
| Опции | defaults, ro, noexec, nosuid, noatime |
| dump | 0=не бэкапить, 1=бэкапить (устарело) |
| fsck | 0=не проверять, 1=первый (root), 2=после root |

```bash
# Применить fstab без перезагрузки
sudo mount -a

# Проверить fstab на ошибки (перед reboot!)
sudo findmnt --verify
```

> **Ошибка в fstab может сделать систему незагружаемой!** Всегда проверяйте перед перезагрузкой.

## Swap (подкачка)

Swap — область на диске, куда ядро выгружает редко используемые страницы памяти.

```bash
# Создать swap-раздел
sudo mkswap /dev/sda3
sudo swapon /dev/sda3

# Создать swap-файл
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Добавить в fstab для автоматической активации
# /swapfile  none  swap  sw  0  0

# Статус
swapon --show
free -h
```

## LVM (Logical Volume Manager)

LVM добавляет уровень абстракции между разделами и файловыми системами: можно изменять размеры «на лету», объединять несколько дисков, делать снапшоты.

### Архитектура

```
Физические диски/разделы
   /dev/sda1   /dev/sdb1
       ↓           ↓
   PV (Physical Volume)        ← pvcreate
       └─────┬─────┘
             ↓
   VG (Volume Group)           ← vgcreate
       ┌─────┼─────┐
       ↓     ↓     ↓
   LV      LV     LV           ← lvcreate
  (root)  (home) (swap)
       ↓     ↓     ↓
   ext4   ext4   swap           ← mkfs / mkswap
```

### Команды

```bash
# Создание
sudo pvcreate /dev/sdb1                    # физический том
sudo vgcreate myvg /dev/sdb1              # группа томов
sudo lvcreate -L 10G -n mydata myvg       # логический том 10 ГБ
sudo lvcreate -l 100%FREE -n rest myvg    # всё оставшееся место
sudo mkfs.ext4 /dev/myvg/mydata           # ФС на LV

# Просмотр
sudo pvs / pvdisplay                      # физические тома
sudo vgs / vgdisplay                      # группы томов
sudo lvs / lvdisplay                      # логические тома

# Расширение (главное преимущество LVM)
sudo lvextend -L +5G /dev/myvg/mydata     # добавить 5 ГБ
sudo resize2fs /dev/myvg/mydata           # расширить ext4
# Для XFS: sudo xfs_growfs /mnt/mydata
```

> LVM-тома доступны как `/dev/VG/LV` (например `/dev/myvg/mydata`) — эти пути можно использовать в fstab и mount.

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| `dd of=/dev/sda` вместо `of=/dev/sdb` | Данные уничтожены. Трижды проверяйте `of=` |
| `umount: target is busy` | Кто-то использует FS. `lsof +D /mnt` → найти процесс |
| Ошибка в fstab → не грузится | Загрузиться с Live USB или `init=/bin/bash` → `mount -o remount,rw /` → исправить |
| `/dev/sda` стал `/dev/sdb` после reboot | Использовать UUID вместо имён устройств |
| `No space left on device` при свободном месте | Кончились inode: `df -i`. Много мелких файлов |
| LVM: lvextend без resize2fs | Том расширен, но FS не знает. Нужен `resize2fs` (ext4) или `xfs_growfs` (XFS) |

## Связанные материалы

- [[linux/explanation/filesystem]] — FHS, типы файлов, пути
- [[linux/how-to/manage-disks]] — практические сценарии
- [[linux/explanation/boot-process]] — как ядро находит корневую FS
