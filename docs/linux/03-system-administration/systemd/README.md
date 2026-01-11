# systemd

## NAME

systemd: современная система инициализации Linux. Управляет сервисами, демонами, загрузкой и логированием системы.

## SYNOPSIS

```bash
systemctl start service              # запустить сервис
systemctl enable service             # включить при загрузке
journalctl -u service                # просмотр логов
sudo systemctl daemon-reload         # перезагрузить конфиги
systemctl list-units --type=target   # список targets
```

## DESCRIPTION

systemd заменил старый SysVinit и стал стандартом в современном Linux (используется в 99% дистрибутивов). Управляет сервисами, демонами, загрузкой системы и встроенным логированием через journald.

## ORGANIZATION

Подраздел состоит из 5 файлов:

| № | Файл | Содержание |
|----|------|-----------|
| 1 | `01-what-is-systemd.md` | Архитектура systemd, история, компоненты |
| 2 | `02-units-services.md` | Создание и управление сервисами |
| 3 | `03-package-management-advanced.md` | Продвинутое управление пакетами |
| 4 | `04-backup-and-recovery.md` | Резервное копирование и восстановление |
| 5 | `05-system-monitoring.md` | Мониторинг системы |

## CONTENTS

### 01-what-is-systemd.md

Полное объяснение systemd:
- История (SysVinit → systemd)
- Архитектура и компоненты
- Units типы (service, target, socket, mount, timer)
- Targets (состояния системы)
- journald (встроенное логирование)
- Boot процесс

### 02-units-services.md

Создание и управление сервисами:
- Service файлы (.service)
- Структура файлов ([Unit], [Service], [Install])
- Примеры сервисов
- systemctl команды
- journalctl логирование
- Troubleshooting

### 03-package-management-advanced.md

Продвинутое управление пакетами:
- Поиск и анализ пакетов
- Разрешение конфликтов
- Версионирование
- Очистка и оптимизация

### 04-backup-and-recovery.md

Резервное копирование и восстановление:
- Стратегии резервного копирования
- Инструменты (rsync, tar, dd)
- Тестирование восстановления

### 05-system-monitoring.md

Мониторинг системы:
- Ресурсы (CPU, RAM, Disk, Network)
- Инструменты мониторинга
- journalctl анализ
- Alerting

## KEY COMMANDS

```bash
# Сервисы
systemctl start service              # запустить
systemctl stop service               # остановить
systemctl restart service            # перезагрузить
systemctl enable service             # включить при загрузке
systemctl disable service            # отключить при загрузке
systemctl status service             # статус

# Логи
journalctl -u service                # логи сервиса
journalctl -f                        # в реальном времени
journalctl -p err                    # только ошибки

# Targets
systemctl list-units --type=target   # все targets
systemctl get-default                # текущий target
systemctl set-default graphical.target # изменить

# Daemon
systemctl daemon-reload              # перезагрузить конфиги
systemctl daemon-reexec              # переинициализировать
```

## RELATED

[[docs/linux/03-system-administration/README|← Back to System Administration]]

## SEE ALSO

- [systemd Official](https://systemd.io/) — официальный сайт
- [systemd Wiki](https://wiki.archlinux.org/title/systemd) — Arch Wiki
- [man systemd](https://man7.org/linux/man-pages/man1/systemd.1.html) — официальная справка
- [man systemctl](https://man7.org/linux/man-pages/man1/systemctl.1.html) — systemctl справка