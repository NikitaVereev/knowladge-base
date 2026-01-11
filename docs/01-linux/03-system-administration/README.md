# System Administration

## NAME

Администрирование Linux систем: управление сервисами, пакетами, пользователями, резервным копированием и мониторингом.

## SYNOPSIS

```bash
systemctl start service                 # управление сервисами
sudo pacman -S package                  # установка пакетов (Arch)
sudo rsync -av source dest              # резервное копирование
journalctl -u service                   # просмотр логов
top                                     # мониторинг системы
```

## DESCRIPTION

Раздел администрирования Linux систем. Охватывает управление сервисами через systemd, продвинутое управление пакетами, резервное копирование, мониторинг и оптимизацию.

Применимо для серверов, рабочих станций и критичных систем.

## ORGANIZATION

Раздел разделён на основной материал и подраздел systemd:

### Основной материал (в главной папке)

| № | Файл | Содержание |
|----|------|-----------|
| 1 | `01-server-basics.md` | Основы серверных систем |
| 2 | `02-package-management-advanced.md` | Продвинутое управление пакетами |
| 3 | `03-user-and-access-management.md` | Управление пользователями и доступом |
| 4 | `04-backup-and-recovery.md` | Резервное копирование и восстановление |
| 5 | `05-system-monitoring.md` | Мониторинг системы |

### systemd подраздел

[[docs/01-linux/03-system-administration/systemd/README|systemd/README.md]] содержит полный раздел по systemd с собственными 5 файлами:
- `01-what-is-systemd.md` — архитектура systemd
- `02-units-services.md` — создание сервисов
- `03-package-management-advanced.md` — управление пакетами
- `04-backup-and-recovery.md` — резервное копирование
- `05-system-monitoring.md` — мониторинг

## CONTENTS

Основной раздел охватывает основные административные задачи. Подраздел systemd углубляется в специфику управления сервисами через современную систему инициализации.

## PREREQUISITES

- Знание Linux основ из [[../../00-foundational/README.md|Foundational]]
- Основные концепции из [[../../01-core-concepts/README.md|Core Concepts]]
- Установленный Linux из [[../../02-distro-specific/README.md|Distro-Specific]]

## RELATED

[[../../00-foundational/README.md|00-foundational]] — Linux основы
[[../../01-core-concepts/README.md|01-core-concepts]] — основные концепции
[[../../02-distro-specific/README.md|02-distro-specific]] — дистрибутивы
