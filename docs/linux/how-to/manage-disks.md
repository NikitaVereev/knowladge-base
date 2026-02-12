---
title: "Управление дисками"
type: how-to
tags: [linux, disks, mount, fstab, fdisk, lvm, partition, filesystem, ext4, swap]
related:
  - "[[linux/explanation/filesystem]]"
  - "[[linux/how-to/monitor-system]]"
  - "[[linux/reference/filesystem-hierarchy]]"
---

# Управление дисками

> **TL;DR:** `lsblk` — список дисков. `mount/umount` — монтирование. `/etc/fstab` — автомонтирование.
> `fdisk` — разметка. `mkfs.ext4` — форматирование. `df -h` — свободное место.

## Просмотр

```bash
lsblk                             # блочные устройства (дерево)
lsblk -f                          # с файловыми системами
df -h                             # свободное место по разделам
sudo fdisk -l                     # подробно о разделах
blkid                             # UUID и типы ФС
```

## Монтирование

```bash
# Монтировать
sudo mount /dev/sdb1 /mnt/data
sudo mount -t ext4 /dev/sdb1 /mnt/data    # указать тип ФС

# USB-флешка
sudo mount /dev/sdc1 /mnt/usb

# ISO-образ
sudo mount -o loop image.iso /mnt/iso

# Размонтировать
sudo umount /mnt/data
# Если "target is busy":
sudo fuser -m /mnt/data          # кто использует?
sudo umount -l /mnt/data         # lazy umount
```

## /etc/fstab (автомонтирование)

```bash
# Формат: device   mountpoint   fstype   options   dump   pass
# Используйте UUID (стабильнее чем /dev/sdX)
blkid /dev/sdb1                  # узнать UUID

# Пример
sudo nano /etc/fstab
```

```
UUID=abc123-def456  /data  ext4  defaults,noatime  0  2
UUID=xyz789-uvw012  none   swap  sw                0  0
```

```bash
# Проверить без перезагрузки
sudo mount -a                    # монтировать всё из fstab
# Если ошибка — исправить fstab! Кривой fstab = система не загрузится.
```

## Разметка и форматирование

```bash
# Разметка (интерактивно)
sudo fdisk /dev/sdb              # MBR
sudo gdisk /dev/sdb              # GPT
sudo cfdisk /dev/sdb             # визуальный

# Форматирование
sudo mkfs.ext4 /dev/sdb1         # ext4 (рекомендуется)
sudo mkfs.xfs /dev/sdb1          # XFS
sudo mkfs.btrfs /dev/sdb1        # Btrfs

# Swap
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
# Добавить в fstab для постоянного использования
```

## Расширение раздела

```bash
# Расширить раздел (если есть свободное место после него)
sudo growpart /dev/sda 2         # расширить partition 2

# Расширить файловую систему
sudo resize2fs /dev/sda2         # ext4
sudo xfs_growfs /                # XFS (по mount point)
```

## Проверка здоровья

```bash
# SMART (нужен smartmontools)
sudo smartctl -a /dev/sda        # полная информация
sudo smartctl -H /dev/sda        # статус здоровья

# Проверка ФС (раздел должен быть размонтирован)
sudo fsck /dev/sdb1
sudo e2fsck -f /dev/sdb1         # ext4
```

## Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| Ошибка в fstab → система не грузится | Recovery mode → `mount -o remount,rw /` → исправить fstab |
| `/dev/sdb` стал `/dev/sdc` после reboot | Использовать UUID в fstab вместо /dev/sdX |
| `target is busy` при umount | `fuser -m /mnt/data`, закрыть процессы, `umount -l` |
| Диск не видно после подключения | `sudo fdisk -l`, `dmesg | tail` |