---
title: "Логирование: journald и syslog"
type: explanation
tags: [linux, logging, journald, syslog, rsyslog, journalctl, systemd, logs]
sources:
  book: "Внутреннее устройство Linux — Брайан Уорд, Глава 7.1–7.2"
  docs: "https://man7.org/linux/man-pages/man1/journalctl.1.html"
related:
  - "[[linux/explanation/systemd]]"
  - "[[linux/reference/systemd-reference]]"
  - "[[linux/how-to/monitor-system]]"
  - "[[linux/reference/cheatsheet]]"
---

# Логирование: journald и syslog

> **TL;DR:** В современных системах `systemd-journald` собирает логи всех сервисов в бинарный журнал. Чтение через `journalctl`. Традиционный syslog (rsyslog, syslog-ng) пишет текстовые файлы в `/var/log/`. Многие системы используют оба: journald собирает, rsyslog пересылает в файлы. Приоритеты сообщений: от `emerg` (0) до `debug` (7).

## Зачем это знать

- «Сервис не работает» → первый шаг всегда `journalctl -u service`
- Диагностика загрузки: `journalctl -b`
- Настройка ротации логов: логи могут заполнить диск
- Понимание потока: приложение → journald/syslog → файл/сеть/агрегатор

## Два подхода к логированию

### systemd-journald (современный)

```
Сервис (stdout/stderr) ──→ journald ──→ бинарный журнал (/run/log/journal/ или /var/log/journal/)
                                  └──→ пересылка в syslog (опционально)
```

- **Бинарный формат** — структурированные записи с метаданными (PID, UID, unit, приоритет, временные метки с микросекундами)
- **Индексирован** — быстрый поиск по любому полю
- **Автоматическая ротация** — по размеру и времени
- **Чтение** — только через `journalctl` (не `cat`/`grep`)

### syslog / rsyslog (традиционный)

```
Сервис → /dev/log (Unix socket) → rsyslog → текстовые файлы в /var/log/
```

- **Текстовый формат** — одна строка = одно сообщение
- **Файлы** — легко читать `cat`, `grep`, `tail -f`
- **Маршрутизация** — по facility (kern, auth, mail...) и severity (err, warning, info...)
- **Пересылка** — легко отправить на удалённый сервер

### Совместная работа

В большинстве дистрибутивов оба работают параллельно: journald собирает логи от systemd-сервисов, rsyslog дополнительно пишет в текстовые файлы `/var/log/`.

## journalctl — чтение журнала

```bash
# Основное
journalctl                          # весь журнал (от старого к новому)
journalctl -e                       # перейти в конец
journalctl -f                       # follow (как tail -f)
journalctl -n 50                    # последние 50 записей

# По сервису
journalctl -u nginx                 # логи конкретного юнита
journalctl -u nginx -u php-fpm     # несколько юнитов
journalctl -u nginx -f             # follow конкретного сервиса

# По времени
journalctl --since "2024-03-01 10:00"
journalctl --since "1 hour ago"
journalctl --since "yesterday" --until "today"
journalctl -S "2024-03-01" -U "2024-03-02"   # короткая форма

# По загрузке
journalctl -b                       # текущая загрузка
journalctl -b -1                    # предыдущая загрузка
journalctl --list-boots             # все записанные загрузки

# По приоритету
journalctl -p err                   # только ошибки и выше
journalctl -p warning               # предупреждения и выше
# Приоритеты: emerg(0) alert(1) crit(2) err(3) warning(4) notice(5) info(6) debug(7)

# По ядру (аналог dmesg)
journalctl -k                       # сообщения ядра
journalctl -k -b                    # ядро с текущей загрузки

# Поиск по тексту
journalctl -g "error|fail"          # grep по содержимому (regex)

# Формат вывода
journalctl -o json-pretty           # JSON (все метаданные)
journalctl -o short-iso             # короткий с ISO-датой
journalctl -o verbose               # все поля записи

# По PID / UID
journalctl _PID=1234
journalctl _UID=1000

# Управление размером
journalctl --disk-usage             # сколько занимает журнал
sudo journalctl --vacuum-size=500M  # сжать до 500 МБ
sudo journalctl --vacuum-time=7d    # удалить старше 7 дней
```

## Конфигурация journald

Файл `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent          # persistent (на диск) / volatile (только RAM) / auto
SystemMaxUse=500M           # максимальный размер журнала на диске
RuntimeMaxUse=100M          # максимальный размер в RAM
MaxFileSec=1month           # ротация по времени
ForwardToSyslog=yes         # пересылать в syslog
```

> `Storage=auto` (по умолчанию): если `/var/log/journal/` существует — пишет на диск, иначе — только в RAM (`/run/log/journal/`, теряется при reboot). Для сохранения логов между перезагрузками: `sudo mkdir -p /var/log/journal`.

## Текстовые логи в /var/log/

| Файл | Содержимое |
|---|---|
| `/var/log/syslog` | Общий лог (Debian/Ubuntu) |
| `/var/log/messages` | Общий лог (RHEL/Fedora) |
| `/var/log/auth.log` | Аутентификация, sudo, SSH (Debian/Ubuntu) |
| `/var/log/secure` | Аутентификация (RHEL/Fedora) |
| `/var/log/kern.log` | Сообщения ядра |
| `/var/log/dmesg` | Загрузка ядра |
| `/var/log/dpkg.log` | Установка/удаление пакетов (apt) |
| `/var/log/yum.log` | Установка/удаление пакетов (yum/dnf) |

```bash
# Просмотр текстовых логов
tail -f /var/log/syslog          # follow
grep "error" /var/log/syslog     # поиск
less /var/log/auth.log           # постраничный просмотр
```

## Приоритеты (severity)

Единая шкала для syslog и journald:

| Число | Имя | Описание |
|---|---|---|
| 0 | `emerg` | Система неработоспособна |
| 1 | `alert` | Требуется немедленное вмешательство |
| 2 | `crit` | Критическая ошибка |
| 3 | `err` | Ошибка |
| 4 | `warning` | Предупреждение |
| 5 | `notice` | Нормальное, но значимое событие |
| 6 | `info` | Информационное |
| 7 | `debug` | Отладочное |

`journalctl -p err` покажет сообщения с приоритетом 0–3 (err и выше).

## Facility (syslog)

Facility определяет источник сообщения. Используется rsyslog для маршрутизации в разные файлы:

| Facility | Описание | Куда обычно пишет |
|---|---|---|
| `kern` | Ядро | `/var/log/kern.log` |
| `auth` / `authpriv` | Аутентификация | `/var/log/auth.log` |
| `mail` | Почтовый сервер | `/var/log/mail.log` |
| `cron` | Планировщик | `/var/log/cron.log` |
| `daemon` | Системные демоны | `/var/log/daemon.log` |
| `local0`–`local7` | Пользовательские | Настраиваемые |

Конфигурация rsyslog: `/etc/rsyslog.conf` и `/etc/rsyslog.d/`. Формат правил: `facility.severity  destination`.

```
# /etc/rsyslog.d/50-default.conf
auth,authpriv.*          /var/log/auth.log
kern.*                   /var/log/kern.log
*.info;mail,auth.none   /var/log/syslog
```

## Ротация логов

### logrotate (для текстовых файлов)

Конфигурация: `/etc/logrotate.conf` и `/etc/logrotate.d/`.

```
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily               # ротация ежедневно
    rotate 14           # хранить 14 файлов
    compress            # сжимать gzip
    delaycompress       # сжимать через одну ротацию
    missingok           # не ругаться если лога нет
    notifempty          # не ротировать пустые
    postrotate          # после ротации
        systemctl reload nginx
    endscript
}
```

### journald (автоматическая ротация)

Настраивается через `SystemMaxUse` и `MaxFileSec` в `journald.conf`, или вручную через `journalctl --vacuum-*`.

## Подводные камни

| Ситуация | Совет |
|----------|-------|
| Логи заполнили диск | `journalctl --vacuum-size=200M` + проверить `/var/log/` |
| `journalctl` пуст после reboot | `Storage=volatile` или нет `/var/log/journal/`. Создать: `sudo mkdir -p /var/log/journal` |
| Нужны логи в текстовом файле | Убедиться что rsyslog установлен и `ForwardToSyslog=yes` |
| Логи приложения не в journald | Приложение не под systemd или пишет в свой файл напрямую |
| `journalctl -u service` пуст | Проверить имя юнита: `systemctl list-units \| grep name` |

## Связанные материалы

- [[linux/explanation/systemd]] — journalctl в контексте systemd
- [[linux/reference/systemd-reference]] — справочник journalctl
- [[linux/how-to/monitor-system]] — мониторинг: CPU, RAM, диск, логи
