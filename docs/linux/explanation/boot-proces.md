---
title: "Процесс загрузки и GRUB"
type: explanation
tags: [linux, boot, grub, uefi, bios, initramfs, kernel, bootloader, secure-boot]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 5"
  docs: "https://www.gnu.org/software/grub/manual/grub/"
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/explanation/architecture]]"
  - "[[linux/explanation/filesystem]]"
  - "[[linux/how-to/arch/install]]"
  - "[[linux/how-to/arch/troubleshooting]]"
---

# Процесс загрузки и GRUB

> **TL;DR:** При включении: BIOS/UEFI → GRUB (выбор ядра) → Kernel (распаковка initramfs, инициализация оборудования) → systemd (PID 1) → default.target. GRUB конфигурируется через `/etc/default/grub` + `update-grub`. Параметры ядра — в `/proc/cmdline`. Аварийная загрузка: `init=/bin/bash` или `rescue.target`.

## Зачем это знать

- Система не загружается — нужно понимать, на каком этапе проблема
- Нужно добавить параметр ядра (например `nomodeset` для GPU)
- Dual-boot с другой ОС
- Восстановление после сбоя (сломан GRUB, потерян пароль root)

## Общая последовательность загрузки

```
┌─────────┐    ┌──────────┐    ┌────────┐    ┌─────────┐    ┌─────────────┐
│BIOS/UEFI│ →  │Bootloader│ →  │ Kernel │ →  │initramfs│ →  │systemd(PID1)│
│  POST   │    │  (GRUB)  │    │        │    │  init   │    │→ targets    │
└─────────┘    └──────────┘    └────────┘    └─────────┘    └─────────────┘
 Firmware       Пользова-       Ядро ини-     Временная      Полная
 находит        тельский        циализи-      корневая FS    инициали-
 загрузочное    выбор ядра      рует обо-     с драйверами   зация
 устройство     и параметров    рудование     для real root   системы
```

## BIOS vs UEFI

| | BIOS (Legacy) | UEFI |
|---|---|---|
| Адресация дисков | MBR (Master Boot Record) | GPT (GUID Partition Table) |
| Размер загрузчика | 441 байт в MBR + gap | Полноценные .efi-файлы |
| Загрузочный раздел | Не нужен отдельный | ESP (EFI System Partition, VFAT) |
| Secure Boot | Нет | Да (проверка подписей) |
| Макс. размер диска | 2 ТБ | 8 ЗБ |

### Загрузка через BIOS + MBR

```
MBR (441 байт) → код в gap между MBR и 1-м разделом → GRUB core → grub.cfg → ядро
```

MBR содержит крошечный загрузчик, который знает только как загрузить следующую часть из gap (промежутка между MBR и первым разделом). Этот промежуток содержит ядро GRUB, которое уже умеет работать с файловыми системами.

### Загрузка через UEFI

```
UEFI firmware → ESP (/boot/efi/EFI/ubuntu/grubx64.efi) → grub.cfg → ядро
```

UEFI напрямую читает VFAT-раздел (ESP) и загружает `.efi`-файл. Каждая ОС имеет свою папку в `/boot/efi/EFI/`.

```bash
# Просмотр UEFI-записей загрузки
efibootmgr -v
# Boot0001* ubuntu  HD(1,GPT,...)/File(\EFI\ubuntu\shimx64.efi)
# Boot0002* Windows HD(1,GPT,...)/File(\EFI\Microsoft\Boot\bootmgfw.efi)
```

### GPT + BIOS (без UEFI)

Для GPT-дисков на BIOS-машинах GRUB требует специальный раздел — **BIOS Boot Partition** (без файловой системы, ~1 МБ), куда записывается код GRUB вместо gap.

### Secure Boot

UEFI может проверять цифровые подписи загрузчика. Цепочка доверия:

```
UEFI → shimx64.efi (подписан Microsoft) → grubx64.efi (подписан дистрибутивом) → ядро
```

Для кастомных ядер или неподписанных модулей может потребоваться отключение Secure Boot в UEFI settings.

## GRUB

### Конфигурация

GRUB генерирует итоговый конфиг автоматически. **Не редактируйте `grub.cfg` напрямую!**

| Файл/каталог | Назначение |
|---|---|
| `/boot/grub/grub.cfg` | Сгенерированный конфиг (не редактировать!) |
| `/etc/default/grub` | Пользовательские настройки (timeout, параметры ядра) |
| `/etc/grub.d/` | Скрипты-генераторы (00_header, 10_linux, 30_os-prober...) |

```bash
# Типичный /etc/default/grub
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
GRUB_DISABLE_OS_PROBER=false
```

```bash
# После изменений — перегенерировать конфиг
sudo update-grub              # Debian/Ubuntu (обёртка)
sudo grub-mkconfig -o /boot/grub/grub.cfg   # универсально
```

### Установка GRUB

```bash
# MBR (весь диск, НЕ раздел!)
sudo grub-install /dev/sda

# UEFI
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

### GRUB CLI (аварийный режим)

Если `grub.cfg` повреждён, GRUB загрузится в CLI (`grub>`). Основные команды:

```
ls                           # список дисков и разделов
ls (hd0,gpt1)/               # содержимое раздела
ls -l (hd0,gpt1)             # UUID и тип FS
echo $root                   # текущий корневой раздел GRUB
set                          # все переменные

# Ручная загрузка ядра
set root=(hd0,gpt2)
linux /boot/vmlinuz-6.8.0 root=UUID=xxx-yyy ro
initrd /boot/initrd.img-6.8.0
boot
```

Именование устройств в GRUB: `(hd0)` = первый диск, `(hd0,msdos1)` = первый MBR-раздел, `(hd0,gpt1)` = первый GPT-раздел.

> **GRUB root ≠ kernel root.** `set root` в GRUB указывает, на каком разделе искать файлы ядра. Параметр `root=` для ядра указывает, какой раздел монтировать как `/`.

### Структура menuentry

```
menuentry 'Ubuntu' {
    search --no-floppy --fs-uuid --set=root xxxxxxxx-xxxx-...
    linux  /boot/vmlinuz-6.8.0 root=UUID=xxxxxxxx-xxxx-... ro quiet splash
    initrd /boot/initrd.img-6.8.0
}
```

### Chainloading (для Windows)

GRUB может передать управление загрузчику другой ОС:

```
menuentry 'Windows' {
    insmod chain
    set root=(hd0,gpt1)
    chainloader +1
}
```

## Параметры ядра

Передаются в строке `linux` в GRUB. Текущие параметры видны в `/proc/cmdline`.

```bash
cat /proc/cmdline
# BOOT_IMAGE=/boot/vmlinuz-6.8.0 root=UUID=xxx ro quiet splash
```

| Параметр | Назначение |
|---|---|
| `root=UUID=xxx` / `root=/dev/sda2` | Корневая FS (UUID надёжнее — не зависит от порядка дисков) |
| `ro` | Монтировать корень read-only (для fsck при загрузке) |
| `quiet` | Минимум сообщений при загрузке |
| `splash` | Графический экран загрузки |
| `nomodeset` | Отключить KMS (если проблемы с GPU) |
| `single` / `1` | Однопользовательский режим (rescue) |
| `init=/bin/bash` | Аварийная оболочка вместо systemd |
| `systemd.unit=rescue.target` | Загрузка в rescue target |

Для временного изменения: в меню GRUB нажать `e`, отредактировать строку `linux`, загрузить `Ctrl+X`.

## initramfs

**initramfs** (initial RAM filesystem) — временная корневая FS, загружаемая в память вместе с ядром. Это cpio-архив (не образ диска!), содержащий:

- Драйверы для доступа к реальному корневому разделу (RAID, LVM, шифрование, нестандартные контроллеры)
- Init-скрипт или мини-systemd, который монтирует реальный корень и переключается на него
- Базовые утилиты для ранней диагностики

```bash
# Посмотреть содержимое
lsinitramfs /boot/initrd.img-$(uname -r)    # Debian/Ubuntu
unmkinitramfs /boot/initrd.img-$(uname -r) /tmp/initrd/

# Пересобрать (после добавления модулей)
sudo update-initramfs -u       # Debian/Ubuntu
sudo mkinitcpio -P             # Arch
sudo dracut --force            # Fedora/RHEL
```

> Если ядро скомпилировано со всеми нужными драйверами встроенными (не модулями), initramfs можно опустить. На практике это редкость.

Термины `initrd` и `initramfs` часто используются как синонимы, хотя технически initrd — устаревший формат (образ диска), а initramfs — современный (cpio-архив).

## Аварийная загрузка

| Метод | Когда использовать | Как |
|---|---|---|
| GRUB edit (`e`) | Нужно изменить параметры ядра | В меню GRUB: `e` → редактировать `linux` → `Ctrl+X` |
| `rescue.target` | systemd загружается, но сервисы сломаны | Параметр: `systemd.unit=rescue.target` |
| `init=/bin/bash` | systemd не работает | Параметр: `init=/bin/bash` (root без пароля, FS read-only → `mount -o remount,rw /`) |
| Live USB | GRUB сломан, диск не загружается | Загрузиться с USB → `mount` → `chroot` → починить |

```bash
# После init=/bin/bash — FS смонтирована read-only
mount -o remount,rw /
# Теперь можно редактировать файлы
passwd root            # сбросить пароль
# Перезагрузка
exec /sbin/init        # или sync && reboot -f
```

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| `grub rescue>` вместо меню | `grub.cfg` не найден. Использовать GRUB CLI для ручной загрузки, затем `update-grub` |
| `error: unknown filesystem` | GRUB не может прочитать раздел. Возможно повреждён суперблок или неверный UUID |
| Пропало меню GRUB (сразу грузит ОС) | `GRUB_TIMEOUT=0` в `/etc/default/grub`. Зажать Shift (BIOS) или Esc (UEFI) при загрузке |
| Dual-boot: Windows не видно | `sudo os-prober` + `GRUB_DISABLE_OS_PROBER=false` + `update-grub` |
| `initramfs` panic: no root | Неверный `root=` или отсутствует драйвер в initramfs. Пересобрать: `update-initramfs -u` |
| Secure Boot мешает | Отключить в UEFI settings или подписать модули ядра |

## Связанные материалы

- [[linux/explanation/systemd]] — что происходит после загрузки ядра
- [[linux/explanation/filesystem]] — FHS, /boot, /etc
- [[linux/how-to/arch/install]] — практика: разметка, GRUB, initramfs
