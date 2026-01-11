---
title: Linux
---

## NAME

Полная документация, справочник и рекомендации по Linux: от основ до специализированных тем в области системного администрирования, конфигурации и использования.

## SYNOPSIS

```
linux/
├── 00-foundational/     # Основы: что такое Linux, установка
├── 01-core-concepts/    # Фундаментальные концепции: файлы, права, процессы
├── 02-distro-specific/  # Дистрибутив-специфичные инструкции
├── 03-system-admin/     # Администрирование: systemd, пользователи, пакеты
├── 04-advanced-topics/  # Продвинутые темы: сеть, безопасность, виртуализация
└── 05-specializations/  # Глубокое погружение: embedded, kernel, HPC
```

## DESCRIPTION

Документация покрывает практическое использование Linux от первой установки до специализированных областей. Каждый раздел самодостаточен и может читаться независимо, но вместе они формируют полный путь от базовых знаний к экспертизе.

**Для кого это:**
- Системные администраторы
- DevOps инженеры
- Разработчики, работающие с Linux
- Люди, готовящиеся к LPIC, Red Hat или другим сертификациям
- Новички, которые устанавливают Linux впервые

**Что вы найдете:**
- Пошаговые инструкции (tutorials и how-to guides)
- Справочную информацию по командам, конфигурациям, файлам
- Объяснение концепций и архитектуры (explanation)
- Практические примеры и решение проблем

## ORGANIZATION

Документация организована от простого к сложному:

| Раздел | Содержание | Начиная с... |
|--------|-----------|--------------|
| **00-foundational** | Что такое Linux, дистрибутивы, установка | Новичков |
| **01-core-concepts** | Файловая система, права доступа, процессы, пакеты | Установленной системы |
| **02-distro-specific** | Arch, Ubuntu/Debian, Fedora/RHEL инструкции | Выбранного дистро |
| **03-system-admin** | systemd, пользователи, мониторинг, резервные копии | Работающей системы |
| **04-advanced-topics** | Сеть, безопасность, производительность, виртуализация | Специализированных тем |
| **05-specializations** | Embedded, kernel development, HPC | Глубокого погружения |

## CONTENTS

### 0️⃣ Foundational
Начните здесь, если вы хотите понять основные концепции.

- **[00-foundational/README.md](docs/01-linux/00-foundational/README.md)** — навигация и содержание
- `01-what-is-linux.md` — определение, история, зачем это нужно
- `02-distributions-guide.md` — сравнение дистрибутивов, выбор
- `03-installation-basics.md` — установка на разные системы

### 1️⃣ Core Concepts
Фундаментальные знания о том, как работает Linux.

- **[01-core-concepts/README.md](docs/01-linux/01-core-concepts/README.md)** — навигация и содержание
- `01-filesystem-hierarchy.md` — структура файловой системы, каталоги
- `02-users-groups-permissions.md` — права доступа, umask, ACL
- `03-processes-and-services.md` — процессы, сигналы, фоновые задачи
- `04-package-management-overview.md` — пакеты, менеджеры, зависимости

### 2️⃣ Distro-Specific
Специфичные инструкции для вашего дистрибутива.

- **[02-distro-specific/README.md](docs/01-linux/02-distro-specific/README.md)** — навигация по дистрибутивам

#### Arch Linux
- `arch-linux/01-installation.md`
- `arch-linux/02-pacman-guide.md` — менеджер пакетов
- `arch-linux/03-aur-guide.md` — Arch User Repository
- `arch-linux/04-maintenance.md` — обновления, чистка
- `arch-linux/05-troubleshooting.md` — решение проблем

#### Ubuntu / Debian
- `ubuntu-debian/01-installation.md`
- `ubuntu-debian/02-apt-guide.md` — менеджер пакетов
- `ubuntu-debian/03-ppa-guide.md` — личные репозитории
- `ubuntu-debian/04-maintenance.md` — обновления, безопасность

### 3️⃣ System Administration
Администрирование работающей системы.

- **[03-system-admin/README.md](./03-system-admin/README.md)** — навигация и содержание

**systemd (специальная подсекция)**
- `systemd/01-what-is-systemd.md` — история, архитектура, почему systemd
- `systemd/02-units-services.md` — юниты, сервисы, таймеры
- `systemd/03-user-services.md` — пользовательские сервисы
- `systemd/04-advanced-systemd.md` — продвинутые возможности, target'ы

**Другие темы**
- `01-user-management.md` — добавление пользователей, группы, sudo
- `02-file-permissions.md` — chmod, chown, специальные биты
- `03-package-management-advanced.md` — поиск, конфликты, зависимости
- `04-backup-and-recovery.md` — резервные копии, восстановление, LVM
- `05-system-monitoring.md` — журналы, мониторинг ресурсов

### 4️⃣ Advanced Topics
Специализированные и продвинутые темы.

- **[04-advanced-topics/README.md](docs/01-linux/04-advanced-topics/README.md)** — навигация и содержание

**Networking**
- `networking/01-networking-basics.md` — IP, маски, маршрутизация
- `networking/02-network-configuration.md` — ifconfig, nmcli, systemd-networkd
- `networking/03-troubleshooting.md` — диагностика сети
- `networking/04-advanced-networking.md` — VLAN, bonding, bridging

**Security & Hardening**
- `security-hardening/01-hardening-guide.md` — базовое усиление безопасности
- `security-hardening/02-firewall-ufw.md` — конфигурация firewall
- `security-hardening/03-encryption.md` — LUKS, SSH, TLS
- `security-hardening/04-audit-compliance.md` — логирование, аудит

**Performance Tuning**
- `performance-tuning/01-monitoring.md` — tools и метрики
- `performance-tuning/02-optimization.md` — оптимизация ОС, дисков
- `performance-tuning/03-profiling.md` — профилирование приложений

**Virtualization**
- `virtualization/01-kvm-guide.md` — KVM/QEMU
- `virtualization/02-containers.md` — Docker, контейнеры
- `virtualization/03-systemd-containers.md` — systemd-nspawn

### 5️⃣ Specializations
Глубокое погружение в специализированные области.

- **[05-specializations/README.md](docs/01-linux/05-specializations/README.md)** — навигация и содержание
- `embedded-linux/` — Linux на встраиваемых системах
- `kernel-development/` — разработка kernel модулей
- `hpc/` — высокопроизводительные вычисления

## NOTES

### Как пользоваться документацией

**Поиск по теме:**
Используйте оглавление выше или перейдите в нужный раздел и откройте его README.

**Линейное чтение:**
Каждый раздел организован логически — читайте файлы в числовом порядке для прогрессивного обучения.

**Перекрестные ссылки:**
Когда файл ссылается на тему из другого раздела, переходите по ссылке для контекста, затем возвращайтесь.

### Структура каждого раздела

Каждая папка содержит:
- **README.md** — навигация и описание раздела
- **01-, 02-, ... документы** — контент в логическом порядке

README каждого раздела объясняет:
- Что именно освещает этот раздел
- Какие предварительные знания нужны
- Порядок чтения файлов
- Связь с другими разделами

### Форматы файлов

Вся документация в **Markdown** (.md). Это позволяет:
- Просматривать на GitHub напрямую
- Читать в любом текстовом редакторе
- Импортировать в Obsidian или другие tools
- Легко версионировать в Git

### Обновления и версионирование

Документация соответствует текущему состоянию Linux (2024-2025). Основной версионный контроль ведется через Git.

Информация, специфичная для версии (например, "в systemd 255+"), указывается явно в тексте.

## SEE ALSO

Связанные документы:
- **Terminal** — документация по shell, tmux, fzf
- **DevOps** — применение Linux в контексте DevOps и облаков
- **Containers** — Docker, Kubernetes, OCI

Внешние ресурсы:
- Linux man pages: `man command-name`
- ArchWiki: https://wiki.archlinux.org
- Debian Wiki: https://wiki.debian.org
