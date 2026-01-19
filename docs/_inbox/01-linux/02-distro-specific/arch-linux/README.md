# Arch Linux

## NAME

Arch Linux: минималистичный дистрибутив с полным контролем и rolling release обновлениями. Официальный archinstall установщик упростил начало работы. Углубленное руководство по установке, pacman, AUR и обслуживанию.

## SYNOPSIS

```
arch-linux/
├── README.md                 # Этот файл
├── 01-installation.md        # archinstall (рекомендуется) + manual (обучение)
├── 02-pacman-guide.md        # Pacman менеджер пакетов
├── 03-aur-guide.md           # AUR репозиторий и пользовательские пакеты
├── 04-maintenance.md         # Обслуживание и оптимизация системы
└── 05-troubleshooting.md     # Решение типичных проблем Arch
```

## DESCRIPTION

Arch Linux — это минималистичный и гибкий дистрибутив Linux, разработанный чтобы быть простым и лёгким. Принцип "KISS" (Keep It Simple, Stupid) означает что вы сами выбираете что устанавливать.

**Особенности:**
- **Rolling Release** — постоянные обновления, не нужно менять версию
- **Простота** — минимум сложности в реализации
- **Полный контроль** — вы решаете что и как установить
- **Свежие пакеты** — всегда последняя версия
- **Отличная документация** — Arch Wiki лучшая в Linux
- **Современная установка** — archinstall официальный установщик с интерактивным меню

**Архитектура установки (2024+):**
- **archinstall** — новый официальный способ
  - Интерактивное меню
  - Автоматическое разбиение диска
  - Выбор пакетов и конфигурации
  
- **Manual** — ручная установка (для обучения и опытных)
  - Полное понимание как работает Arch
  - Полный контроль над каждым шагом
  - Обучающий процесс

**Когда выбирать Arch (с archinstall):**
- ✓ Хотите полный контроль (но не обязательно ручная установка)
- ✓ Интересуетесь "как это работает"
- ✓ Нравятся свежие версии пакетов
- ✓ Хотите современный, активно разрабатываемый дистрибутив

**Когда НЕ выбирать Arch:**
- ✗ Нужна максимальная стабильность (выбирайте Debian Stable)
- ✗ Нет времени на обновления (Arch требует регулярного обслуживания)
- ✗ Нужна долгая поддержка версии (LTS) — выбирайте Ubuntu LTS

## PREREQUISITES

**Знание:**
- Основы Linux (файловая система, команды, права)
- Понимание как работают процессы и системные демоны
- Знакомство с менеджерами пакетов (apt/dnf/pacman базовое)

**Навыки:**
- Работать в терминале (не обязательно опытно)
- Редактировать конфигурационные файлы (nano/vim — базовое)
- Понимать что такое partition и filesystem (базовое)

**Оборудование:**
- USB флешка (для установки)
- Не менее 20 GB свободного места на диске
- Процессор и RAM (минимум 512 MB RAM, рекомендуется 2+ GB)
- Доступ в интернет (кабель или WiFi)

## ORGANIZATION

| № | Файл | Содержание | Тип |
|---|------|-----------|------|
| 1 | `01-installation.md` | Archinstall (основной) + manual (опционально) | Tutorial |
| 2 | `02-pacman-guide.md` | Pacman менеджер пакетов | Reference + How-to |
| 3 | `03-aur-guide.md` | AUR репозиторий и yaourt/yay | How-to |
| 4 | `04-maintenance.md` | Обслуживание и оптимизация | How-to |
| 5 | `05-troubleshooting.md` | Решение типичных проблем | Reference |

## CONTENTS

### 1. Installation
**[01-installation.md](docs/_inbox/01-linux/02-distro-specific/arch-linux/01-installation.md)** | Type: **Tutorial**

Установка Arch Linux двумя способами:

**Способ 1: Archinstall (РЕКОМЕНДУЕТСЯ)**
- Официальный интерактивный установщик
- Меню для выбора параметров
- Автоматическое разбиение диска
- Выбор пакетов и конфигурации

**Способ 2: Manual (ОПЦИОНАЛЬНО)**
- Ручная установка для обучения
- Пошаговое разбиение диска
- Форматирование и монтирование
- Установка bootloader

### 2. Pacman Guide
**[02-pacman-guide.md](02-pacman-guide.md)** | Type: **Reference + How-to**

Полное руководство по pacman — пакетному менеджеру Arch Linux. Более мощный и быстрый чем apt.

Содержит:
- Базовые команды (install, remove, upgrade)
- Поиск пакетов (search, info)
- Работа с группами пакетов (groups)
- Работа с кэшем (cache cleaning)
- Конфигурация pacman
- Makepkg для сборки из исходников
- Оптимизация производительности

→ Используйте как справку для pacman команд

### 3. AUR Guide
**[03-aur-guide.md](03-aur-guide.md)** | Type: **How-to**

Arch User Repository (AUR) — как использовать пользовательские пакеты. AUR содержит пакеты которых нет в официальных репозиториях.

Содержит:
- Что такое AUR и как он работает
- Поиск пакетов в AUR
- Установка пакетов из AUR
- Yaourt и yay helper'ы (упрощают работу)
- Безопасность при использовании AUR
- Создание собственного пакета (advanced)

→ Используйте когда нужна программа которой нет в pacman

### 4. Maintenance
**[04-maintenance.md](docs/_inbox/01-linux/02-distro-specific/arch-linux/04-maintenance.md)** | Type: **How-to**

Обслуживание и оптимизация Arch Linux. Что делать после установки чтобы система работала хорошо.

Содержит:
- Обновление системы (ежедневное)
- Очистка кэша и orphan пакетов
- Оптимизация boot времени
- Мониторинг системы
- Управление сервисами
- Резервные копии
- Оптимизация производительности

→ Прочитайте чтобы система работала на отлично

### 5. Troubleshooting
**[05-troubleshooting.md](docs/_inbox/01-linux/02-distro-specific/arch-linux/05-troubleshooting.md)** | Type: **Reference**

Решение типичных проблем Arch Linux. Что делать когда что-то пошло не так.

Содержит:
- Система не загружается
- Нет интернета
- Пакет конфликтует с другим
- Pacman зависает
- Нет звука или видео
- Вход в GRUB при ошибке
- Восстановление системы
- Откат обновлений

→ Используйте когда что-то не работает

## LEARNING PATH

**Первая установка (archinstall путь):**

1. Прочитайте README (этот файл) 
2. Посмотрите 01-installation.md (архистол раздел)
3. Установите Arch через archinstall
4. Обновите систему первый раз
5. Прочитайте 02-pacman-guide.md
6. Установите пару программ через pacman
7. Прочитайте 03-aur-guide.md
8. (опционально) Попробуйте установить из AUR
9. Прочитайте 04-maintenance.md
10. Выполните рекомендации maintenance

- Используйте 01-installation.md как шпаргалку для archinstall
- 02-pacman-guide.md как справку для pacman
- 03-aur-guide.md если нужна нестандартная программа
- 04-maintenance.md опционален (опытные знают)
- 05-troubleshooting.md при проблемах

## RELATED

Следующие шаги:
- [[docs/_inbox/01-linux/02-distro-specific/README|Distro-Specific Index]] — другие дистрибутивы (Ubuntu и т.д.)
- [[docs/_inbox/01-linux/03-system-administration/README|System Administration]] — администрирование сервера

Предыдущие разделы:
- [[docs/_inbox/01-linux/01-core-concepts/README|Core Concepts]] — основные концепции Linux
- [[docs/_inbox/01-linux/00-foundational/README|Foundational]] — основы Linux

## NOTES

- **Arch больше не означает сложность** — archinstall сделал это доступным
- **Archinstall рекомендуется** — для первой установки используйте именно его
- **Manual остаётся для обучения** — если хотите понять как работает система
- **Архив требует обновлений** — даже если что-то работает, обновляйте регулярно
- **Pacman не сломает систему** — это стабильный и надежный менеджер
- **AUR требует доверия** — проверяйте PKGBUILD перед установкой
- **Arch = современный дистрибутив** — всегда новейшие версии и технологии
- **Wiki предоставляет все ответы** — Arch Wiki должна быть вашим лучшим другом

## SEE ALSO

Официальные ресурсы:

- [Arch Linux Website](https://archlinux.org/) — основной сайт
- [Arch Linux Wiki](https://wiki.archlinux.org/) — ЛУЧШАЯ документация для всех
- [Archinstall Guide](https://wiki.archlinux.org/title/Archinstall) — официальное руководство по archinstall ⭐
- [Installation Guide](https://wiki.archlinux.org/title/Installation_guide) — полное руководство для manual
- [Pacman](https://wiki.archlinux.org/title/Pacman) — подробно про pacman
- [AUR](https://aur.archlinux.org/) — сам репозиторий
- [Arch Philosophy](https://wiki.archlinux.org/title/The_Arch_Way) — философия Arch

Инструменты:

- [archinstall](https://wiki.archlinux.org/title/Archinstall) — официальный интерактивный установщик ⭐
- [yay](https://github.com/Jguer/yay) — AUR helper (рекомендуется)
- [Arch Linux Forums](https://bbs.archlinux.org/) — форум сообщества
- [Arch Linux Subreddit](https://www.reddit.com/r/archlinux/) — Reddit сообщество