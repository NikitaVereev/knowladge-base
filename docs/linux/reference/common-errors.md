---
title: "Справочник: Типичные ошибки Linux"
type: reference
tags:
  - linux
  - errors
  - troubleshooting
  - diagnostics
  - reference
sources:
  book: Внутреннее устройство Linux — Брайан Уорд
related:
  - "[[linux/reference/cheatsheet]]"
  - "[[network-diagnostics]]"
  - "[[linux/how-to/monitor-system]]"
  - "[[linux/explanation/permissions-model]]"
  - "[[linux/explanation/networking]]"
---

# Справочник: Типичные ошибки Linux

> Каталог частых ошибок, которые выдаёт система, и что с ними делать. Организован по подсистемам.

## Структура сообщения об ошибке

```
программа: описание_проблемы: системная_ошибка
```

Пример: `cp: cannot create regular file '/root/file': Permission denied` — программа `cp`, действие `cannot create regular file`, причина `Permission denied`.

> **Warning vs Error.** Warning — предупреждение, программа продолжает работу. Error — ошибка, операция не выполнена. Код возврата: `0` = успех, `> 0` = ошибка. Проверка: `echo $?` после команды.

## Файловая система

| Ошибка | Причина | Решение |
|---|---|---|
| `Permission denied` | Нет прав (rwx) на файл/каталог | `ls -la file` → `chmod`/`chown` или `sudo` |
| `No such file or directory` | Файл не существует или ошибка в пути | Проверить путь, `ls` родительского каталога |
| `Not a directory` | Путь содержит файл вместо каталога | `ls -la` каждого компонента пути |
| `Is a directory` | Попытка записать в каталог как в файл | Указать имя файла внутри каталога |
| `File exists` | Файл уже существует (например `mkdir`) | Использовать `-p` для mkdir, проверить имя |
| `No space left on device` | Диск заполнен | `df -h` → найти и очистить. Если место есть — кончились inode: `df -i` |
| `Read-only file system` | FS смонтирована read-only | `mount -o remount,rw /` (если можно) |
| `Too many open files` | Процесс исчерпал лимит FD | `ulimit -n` → увеличить в `/etc/security/limits.conf` |
| `Device or resource busy` | Файл/FS используется | `lsof +D /path` или `fuser /path` → найти процесс |
| `Structure needs cleaning` | ФС повреждена | `sudo fsck /dev/sdXN` (раздел должен быть размонтирован) |

## Права доступа

| Ошибка | Причина | Решение |
|---|---|---|
| `Operation not permitted` | Нужны привилегии root (или capability) | `sudo` или проверить права/владельца |
| `Permission denied` при exec | Нет бита execute | `chmod +x script.sh` |
| `sudo: user is not in the sudoers file` | Пользователь не в группе sudo | `usermod -aG sudo user` (от root) |
| `Authentication failure` (sudo) | Неверный пароль | Ввести пароль **текущего** пользователя, не root |

## Процессы и память

| Ошибка / Симптом | Причина | Решение |
|---|---|---|
| `Killed` (OOM) | Out of Memory Killer убил процесс | Добавить RAM/swap, проверить утечки: `dmesg \| grep -i oom` |
| `Segmentation fault` | Программа обратилась к чужой памяти | Баг в программе. Обновить или сообщить разработчикам |
| `Cannot allocate memory` | RAM + swap исчерпаны | `free -h` → добавить swap или найти утечку |
| `fork: retry: Resource temporarily unavailable` | Лимит процессов | `ulimit -u` → увеличить nproc, проверить fork bomb |
| Zombie-процессы (`Z` в ps) | Parent не вызвал `waitpid()` | Не критично (не занимают ресурсы). Убить parent для очистки |
| High load, but CPU idle | Много D-state процессов | Проблема с диском/NFS: `iostat`, `iotop` |

## Сеть

| Ошибка | Причина | Решение |
|---|---|---|
| `Connection refused` | Порт закрыт или сервис не запущен | `ss -tlnp \| grep PORT` → запустить сервис |
| `Connection timed out` | Пакеты теряются (firewall, маршрут) | Проверить firewall: `iptables -L`, маршрут: `ip route get IP` |
| `No route to host` | Нет маршрута до хоста | `ip route` → проверить gateway |
| `Network is unreachable` | Нет default route | `ip route add default via GATEWAY` |
| `Name or service not known` | DNS не может разрешить имя | `ping 8.8.8.8` (если работает — проблема DNS). `cat /etc/resolv.conf` |
| `bind: Address already in use` | Порт занят другим процессом | `ss -tlnp \| grep PORT` → остановить или сменить порт |
| `Host key verification failed` | SSH-ключ хоста изменился | Удалить строку из `~/.ssh/known_hosts` (если смена ключа ожидаема) |

## Пакетные менеджеры

| Ошибка | Причина | Решение |
|---|---|---|
| `Unable to locate package` | Пакет не найден | `sudo apt update` → повторить, проверить имя |
| `dpkg was interrupted` | Прерванная установка | `sudo dpkg --configure -a` |
| `Unmet dependencies` | Конфликт зависимостей | `sudo apt --fix-broken install` |
| `Could not get lock /var/lib/dpkg/lock` | apt уже запущен | Подождать. Если зависло: `sudo kill PID` соответствующего процесса |
| `GPG error: NO_PUBKEY` | Нет ключа репозитория | Импортировать ключ по инструкции репозитория |

## systemd / Сервисы

| Ошибка / Симптом | Причина | Решение |
|---|---|---|
| `Unit not found` | Юнит не существует | Проверить имя: `systemctl list-unit-files \| grep name` |
| `Failed to start` → `exit-code` | Ошибка в программе | `journalctl -u service -n 50` → смотреть причину |
| `status=203/EXEC` | Нет файла или нет прав | Проверить `ExecStart` путь и `chmod +x` |
| `status=217/USER` | Пользователь в `User=` не существует | `id username` → создать пользователя |
| Сервис enabled, не запускается | `enable` ≠ `start` | `systemctl enable --now service` |
| Изменения в unit-файле не применились | Нужен daemon-reload | `sudo systemctl daemon-reload` |

## Загрузка

| Симптом | Причина | Решение |
|---|---|---|
| `grub rescue>` | GRUB не находит конфиг | Ручная загрузка из GRUB CLI, затем `update-grub` |
| `Kernel panic` | Ядро не может смонтировать root | Проверить `root=` в параметрах ядра, пересобрать initramfs |
| Emergency mode | Ошибка в fstab | Исправить `/etc/fstab`, `mount -a` для проверки |

## Диагностика: куда смотреть

```bash
# 1. Последние ошибки в журнале
journalctl -p err -b               # ошибки с текущей загрузки

# 2. Логи конкретного сервиса
journalctl -u nginx -n 50

# 3. Ядро
dmesg | tail -30
journalctl -k

# 4. Код возврата последней команды
echo $?                            # 0 = OK, >0 = ошибка

# 5. Отладка
strace -o trace.log command        # системные вызовы
```

## Связанные материалы

- [[linux/reference/cheatsheet]] — команды для диагностики
- [[linux/how-to/network-diagnostics]] — сетевая диагностика
- [[linux/how-to/monitor-system]] — мониторинг CPU, RAM, диска
