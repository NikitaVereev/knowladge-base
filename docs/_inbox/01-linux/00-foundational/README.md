# Foundational

## NAME

Основы Linux: определение, история, выбор дистрибутива и установка системы.

## SYNOPSIS

```
foundational/
├── 01-what-is-linux.md         # История, архитектура, компоненты
├── 02-distributions-guide.md   # Сравнение и выбор дистрибутива
└── 03-installation-basics.md   # Установка на разные платформы
```

## DESCRIPTION

Этот раздел охватывает базовые знания о Linux для тех, кто с ним еще не знаком. Вы узнаете, что такое Linux, чем отличаются дистрибутивы, как выбрать подходящий и как его установить.

**Практическое применение:**
- Понять основные концепции и архитектуру Linux
- Выбрать подходящий дистрибутив для ваших задач
- Успешно установить Linux на вашу машину
- Получить работающую систему для дальнейшего обучения

## PREREQUISITES

**Знание:**
- Базовое понимание операционных систем (Windows, macOS, или другие)
- Что такое командная строка (command line)
- Основные файловые операции (копирование, перемещение файлов)

**Оборудование:**
- Компьютер (ноутбук, десктоп, или виртуальная машина)
- Для физической установки: USB флешка или виртуализация software (VirtualBox, QEMU)
- Доступ в интернет для загрузки ISO образа

## ORGANIZATION

Этот раздел состоит из:

| № | Файл | Содержание | Тип |
|---|------|-----------|------|
| 1 | `01-what-is-linux.md` | Определение, история, архитектура, компоненты | Explanation |
| 2 | `02-distributions-guide.md` | Сравнение дистрибутивов, критерии выбора | How-to |
| 3 | `03-installation-basics.md` | Установка: виртуальная машина, USB, dual-boot, bare metal | Tutorial |

## CONTENTS

### 1. What is Linux?
**[01-what-is-linux.md](01-what-is-linux.md)** | Type: **Explanation**

История Linux, определение kernel и distribution, архитектура компонентов системы, файловая система и современное состояние.

Содержит:
- Linux kernel vs distribution
- История создания и GPL лицензия
- Архитектура: слои от hardware до приложений
- Стандартная файловая система (FHS)
- Почему Linux популярен

→ Прочитайте первым, это дает контекст для остального

### 2. Choosing a Distribution
**[02-distributions-guide.md](02-distributions-guide.md)** | Type: **How-to**

Сравнение основных дистрибутивов (Debian, Red Hat, Arch) по критериям: простота установки, частота обновлений, сообщество, документация, поддержка.

Содержит:
- Три главных семейства (Debian, Red Hat, Arch)
- Популярные дистрибутивы в каждом семействе
- LTS vs Rolling Release
- Таблица выбора по критериям
- Команды для определения установленного дистро
- Решение проблем при выборе

→ Прочитайте после 01, когда поймете архитектуру

### 3. Installation Basics
**[03-installation-basics.md](03-installation-basics.md)** | Type: **Tutorial**

Пошаговое руководство по установке Linux. Охватывает основные способы: VirtualBox, Live USB, dual-boot, bare metal.

Содержит:
- Четыре способа установки с плюсами/минусами
- Как загрузиться с USB
- Создание USB образа (Windows Rufus, macOS/Linux dd)
- Команды после установки (обновление, установка ПО)
- Частые проблемы и решения
- Важные директории Linux

→ Используйте для практической установки системы

## LEARNING PATH

Рекомендуемый порядок:

1. **01-what-is-linux.md** — поймите архитектуру (15-20 мин)
2. **02-distributions-guide.md** — выберите дистрибутив (10-15 мин)
3. **03-installation-basics.md** — установите систему (30-60 мин на установку)

После этого раздела вы будете готовы перейти к [[docs/_inbox/01-linux/01-core-concepts/README|Core Concepts]].

## RELATED

Связанные разделы после этого:
- [[docs/_inbox/01-linux/01-core-concepts/README|Core Concepts]] — фундаментальные знания о файлах, правах, процессах
- [[docs/_inbox/01-linux/02-distro-specific/README|Distro-Specific]] — углубленные инструкции для вашего дистрибутива

Контекст:
- [[docs/_inbox/01-linux/index|Main documentation]] — полный каталог всей документации

## NOTES

- **Примеры используют разные дистрибутивы** — выбранный вами дистрибутив может иметь небольшие различия в названиях команд
- **Установка может занять время** — установщик может обновляться в фоне, это нормально
- **Резервная копия важна** — если устанавливаете на реальный диск с данными, сделайте backup

## SEE ALSO

Дополнительные ресурсы:
- [Linux Foundation](https://www.linuxfoundation.org/) — официальная информация о Linux
- [DistroWatch](https://distrowatch.com) — каталог и рейтинг дистрибутивов
- [Wikipedia: Linux](https://en.wikipedia.org/wiki/Linux) — исторический контекст
- [Linux.com](https://www.linux.com/) — новости и статьи

Официальные сайты дистрибутивов:
- [archlinux.org](https://www.archlinux.org)
- [ubuntu.com](https://ubuntu.com)
- [debian.org](https://www.debian.org)
- [fedoraproject.org](https://fedoraproject.org)

Утилиты для установки:
- [Rufus](https://rufus.ie) — создание загрузочной USB на Windows
- [VirtualBox](https://virtualbox.org) — виртуализация
- [QEMU](https://www.qemu.org) — альтернатива VirtualBox
