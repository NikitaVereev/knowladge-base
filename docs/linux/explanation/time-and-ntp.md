---
title: "Время, часовые пояса и NTP"
type: explanation
tags: [linux, time, ntp, timezone, timedatectl, chrony, hwclock, rtc, utc]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 7.4"
  docs: "https://man7.org/linux/man-pages/man1/timedatectl.1.html"
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/how-to/schedule-tasks]]"
  - "[[linux/how-to/recipes/initial-server-setup]]"
  - "[[linux/reference/cheatsheet]]"
---

# Время, часовые пояса и NTP

> **TL;DR:** Linux хранит два часов: аппаратные (RTC, в BIOS) и системные (в ядре). Серверы держат RTC в UTC. Синхронизация по NTP через `systemd-timesyncd` (по умолчанию) или `chrony` (точнее). `timedatectl` — единая команда для управления всем: время, timezone, NTP-статус.

## Зачем это знать

- Логи, cron, сертификаты TLS, Kerberos, двухфакторная аутентификация — всё зависит от точного времени
- Рассинхрон в несколько минут может сломать аутентификацию в AD/LDAP
- В кластерах (Kubernetes, распределённые БД) расхождение часов приводит к потере данных
- Dual-boot с Windows: конфликт UTC vs localtime на аппаратных часах

## Два часовых механизма

### RTC (Real-Time Clock) — аппаратные часы

Микросхема на материнской плате с батарейкой. Тикает даже при выключенном компьютере. Ядро читает RTC при загрузке для инициализации системных часов.

```bash
# Прочитать аппаратные часы
sudo hwclock --show
# 2024-03-15 12:00:00.000000+03:00

# Записать системное время в RTC
sudo hwclock --systohc

# Прочитать RTC и установить системное время
sudo hwclock --hctosys
```

### System clock — системные часы

Поддерживаются ядром в оперативной памяти. Работают только пока система включена. Считают время в секундах с Unix Epoch (1 января 1970 00:00:00 UTC).

```bash
date                          # текущее системное время
date +%s                      # Unix timestamp (секунды с epoch)
date -u                       # время в UTC
date -d @1710500000           # timestamp → дата
```

## UTC vs Local Time

**UTC** (Coordinated Universal Time) — единая шкала времени без часовых поясов.

| | RTC в UTC | RTC в localtime |
|---|---|---|
| Серверы | ✓ Стандарт | Не используется |
| Linux-only | ✓ Рекомендуется | Допустимо |
| Dual-boot с Windows | Конфликт* | Windows ожидает localtime |

> *Windows по умолчанию считает RTC = localtime. При dual-boot либо переключить Windows на UTC (`reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1`), либо Linux на localtime (`timedatectl set-local-rtc 1` — не рекомендуется).

## timedatectl — управление всем

```bash
# Полный статус
timedatectl
#                Local time: Fri 2024-03-15 15:00:00 MSK
#            Universal time: Fri 2024-03-15 12:00:00 UTC
#                  RTC time: Fri 2024-03-15 12:00:00
#                 Time zone: Europe/Moscow (MSK, +0300)
# System clock synchronized: yes          ← NTP работает
#               NTP service: active       ← timesyncd/chrony запущен
#           RTC in local TZ: no           ← RTC в UTC (правильно)

# Часовой пояс
timedatectl list-timezones              # все доступные
timedatectl list-timezones | grep Europe
timedatectl set-timezone Europe/Moscow  # установить

# NTP
timedatectl set-ntp true                # включить NTP-синхронизацию
timedatectl set-ntp false               # отключить

# Ручная установка (только при выключенном NTP)
timedatectl set-time "2024-03-15 15:00:00"
```

### Часовые пояса под капотом

Timezone определяется символической ссылкой `/etc/localtime` → `/usr/share/zoneinfo/<зона>`:

```bash
ls -la /etc/localtime
# /etc/localtime -> /usr/share/zoneinfo/Europe/Moscow

# Альтернативно: переменная окружения TZ
TZ=America/New_York date     # показать время в Нью-Йорке
```

## NTP — синхронизация по сети

NTP (Network Time Protocol) синхронизирует системные часы с эталонными серверами. Точность — миллисекунды.

### systemd-timesyncd (по умолчанию)

Лёгкий SNTP-клиент, встроен в systemd. Достаточен для большинства случаев.

```bash
# Статус
timedatectl timesync-status
# Server: 185.125.190.58 (ntp.ubuntu.com)
# Poll interval: 32s → 2048s (адаптивно)
# Offset: +2.5ms

# Конфигурация: /etc/systemd/timesyncd.conf
# [Time]
# NTP=ntp.ubuntu.com
# FallbackNTP=0.pool.ntp.org 1.pool.ntp.org

systemctl status systemd-timesyncd
```

### chrony (продвинутый)

Точнее, быстрее адаптируется после спящего режима и при нестабильном соединении. Рекомендуется для серверов и виртуальных машин.

```bash
# Установка
sudo apt install chrony       # Debian/Ubuntu
sudo pacman -S chrony         # Arch

# Конфигурация: /etc/chrony.conf (или /etc/chrony/chrony.conf)
# server ntp.ubuntu.com iburst
# pool 2.pool.ntp.org iburst

# Управление
systemctl status chronyd
chronyc tracking              # текущая точность и источник
chronyc sources -v            # все NTP-серверы и их статус
chronyc makestep              # принудительная коррекция (если рассинхрон большой)
```

> **timesyncd vs chrony:** они конфликтуют. При установке chrony обычно отключает timesyncd. Не запускайте оба одновременно.

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| «System clock synchronized: no» | `timedatectl set-ntp true`, проверить firewall (UDP 123) |
| Время скачет после reboot | RTC в другом формате (UTC/localtime). `timedatectl set-local-rtc 0` |
| Dual-boot: Windows сбивает время | Переключить Windows на UTC или Linux на localtime |
| Cron запускается не в то время | Проверить timezone: `timedatectl`, cron работает в локальном времени |
| Сертификат «not yet valid» | Часы отстают. Синхронизировать по NTP |
| В контейнере нет доступа к RTC | Контейнер использует часы хоста. Timezone через переменную `TZ` или mount `/etc/localtime` |

## Связанные материалы

- [[linux/explanation/systemd]] — timedated как часть systemd
- [[linux/how-to/schedule-tasks]] — cron и systemd timers зависят от точного времени
- [[linux/how-to/recipes/initial-server-setup]] — настройка timezone при установке сервера
