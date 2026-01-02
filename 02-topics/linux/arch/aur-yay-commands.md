---
created: 2026-01-01
tags: [arch-linux, aur, yay]
type: note
---

# AUR и yay - полный справочник

## Основная идея

**AUR** (Arch User Repository) - репозиторий пользовательских пакетов (не официальные).

**yay** - помощник для работы с AUR (вроде pacman но + AUR).

**Когда нужен AUR:**
- Пакета нет в официальных репозиториях
- Программа новая или нишевая
- Patched версия приложения
- Бета версии программ

**Когда использовать pacman:**
- Стабильные, официальные пакеты
- Большинство программ
- Системные инструменты

---

## ЧАСТЬ 1: Установка yay

### Требования перед установкой

```bash
# 1. Установить git
sudo pacman -S git

# 2. Установить base-devel (для сборки пакетов)
sudo pacman -S base-devel

# base-devel включает:
# - gcc (компилятор)
# - make (система сборки)
# - patch (утилиты)
# - pkg-config (конфигурация)
```

### Стандартная установка yay

```bash
# 1. Клонировать репозиторий yay
git clone https://aur.archlinux.org/yay.git

# 2. Перейти в папку
cd yay

# 3. Собрать и установить
makepkg -si

# Что это делает:
# makepkg = собрать пакет из PKGBUILD файла
# -s = установить зависимости автоматически
# -i = установить после сборки

# 4. (Опционально) Удалить папку после установки
cd ..
rm -rf yay
```

### Что происходит при установке yay

```
git clone https://aur.archlinux.org/yay.git
  ↓
Скачивается исходный код yay

cd yay
  ↓
Переход в папку с PKGBUILD

makepkg -si
  ↓
Читает PKGBUILD файл (инструкции по сборке)
  ↓
Скачивает исходный код yay
  ↓
Компилирует (gcc компилятор)
  ↓
Создает пакет .tar.zst
  ↓
Устанавливает через pacman
```

### Проверить что установлен

```bash
# Проверить версию yay
yay --version

# Проверить что yay работает
yay -Ps
# Покажет статистику системы
```

### Альтернативные AUR помощники

```bash
# Если не нравится yay, есть другие:

# paru (аналог yay, написан на Rust)
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si

# pacaur (старый, но стабильный)
yay -S pacaur

# auracle (легковесный)
yay -S auracle
```

---

## ЧАСТЬ 2: Установка пакетов из AUR

### Простая установка

```bash
# Установить пакет из AUR (как pacman -S но с AUR)
yay package_name

# Примеры:
yay visual-studio-code-bin
yay discord
yay spotify
```

### Что происходит при yay install

```
yay package_name
  ↓
yay ищет пакет в AUR
  ↓
Показывает информацию (описание, зависимости)
  ↓
Asks: "Proceed with installation? [Y/n]"
  ↓
Скачивает PKGBUILD файл с инструкциями
  ↓
Проверяет зависимости
  ↓
Устанавливает зависимости через pacman
  ↓
Компилирует пакет (может занять время!)
  ↓
Устанавливает собранный пакет
  ↓
Готово!
```

### Установка без интерактивности

```bash
# Установить без вопросов (для скриптов)
yay -S --noconfirm package_name

# Все флаги вместе:
yay -S --noconfirm --needed package1 package2

# --needed = не переустанавливать если уже есть
```

### Установка с редактированием PKGBUILD

```bash
# Некоторые пакеты требуют редактирования перед установкой
yay package_name

# yay предложит отредактировать PKGBUILD перед сборкой
# Нажмите Enter для редактирования в $EDITOR (nano, vim и т.д.)

# Это нужно когда:
# - В PKGBUILD есть deprecated код
# - Нужны специальные флаги сборки
# - Есть warning'и
```

---

## ЧАСТЬ 3: Обновление пакетов

### Обновить всё (pacman + AUR)

```bash
# Обновить ВСЕ пакеты (официальные + AUR)
yay -Syu

# Эквивалент pacman -Syu но + AUR пакеты
```

### Обновить только AUR пакеты

```bash
# Обновить ТОЛЬКО из AUR (не официальные)
yay -Sua

# Полезно когда официальные пакеты часто обновляются
# А вы хотите контролировать их отдельно
```

### Обновить без переустановки

```bash
# Посмотреть что можно обновить
yay -Qu

# Показывает список без установки
```

### Интерактивное обновление (выбор какие обновлять)

```bash
# Обновить с выбором какие пакеты (если yay это поддерживает)
yay -Syu --ask 4

# Спросит для каждого пакета: обновить? (Y/n)
```

---

## ЧАСТЬ 4: Поиск пакетов

### Поиск в AUR

```bash
# Поиск пакета в AUR
yay -Ss keyword

# Примеры:
yay -Ss visual-studio-code
yay -Ss discord
yay -Ss electron

# yay ищет везде (и официальные и AUR)
# Показывает из какого репо каждый пакет
```

### Поиск только установленных

```bash
# Поиск среди установленных пакетов
yay -Qs keyword

# Примеры:
yay -Qs visual-studio-code
yay -Qs python

# Показывает установленные пакеты с этим словом
```

### Информация о пакете из AUR

```bash
# Полная информация о пакете из AUR
yay -Si package_name

# Показывает:
# - Описание
# - Версию
# - Зависимости
# - Размер при сборке
# - URL
# - Лицензию

# Примеры:
yay -Si visual-studio-code-bin
yay -Si spotify
```

### Информация об установленном пакете

```bash
# Информация об установленном пакете
yay -Qi package_name

# Показывает:
# - Версию
# - Размер
# - Откуда установлен (AUR или официальный репо)
# - Зависимости
# - Когда установлен

# Примеры:
yay -Qi yay
yay -Qi firefox
```

### Посмотреть файлы пакета

```bash
# Какие файлы установил пакет
yay -Ql package_name

# Примеры:
yay -Ql yay
yay -Ql discord | grep bin

# Полезно для поиска конфигов или бинарников
```

### Найти какому пакету принадлежит файл

```bash
# Обратный поиск: какой пакет установил файл?
yay -Qo /usr/bin/yay

# Результат:
# /usr/bin/yay is owned by yay 12.0.0-1
```

---

## ЧАСТЬ 5: Управление AUR пакетами

### Список пакетов из AUR

```bash
# Показать ВСЕ пакеты установленные из AUR
yay -Qm

# Результат: список пакетов с их версиями
# Пакеты которые не из официальных репо

# Примеры:
yay -Qm | grep -i visual
# Покажет VS Code и подобные из AUR
```

### Список только официальных пакетов

```bash
# Показать ВСЕ пакеты из официальных репо
yay -Qn

# Противоположность -Qm
# Не включает AUR пакеты
```

### Удалить пакет из AUR

```bash
# Удалить пакет (как pacman -R)
yay -R package_name

# С зависимостями:
yay -Rs package_name

# С конфигами:
yay -Rns package_name

# Примеры:
yay -Rns discord
yay -Rns visual-studio-code-bin
```

---

## ЧАСТЬ 6: Очистка и оптимизация

### Очистить ненужные зависимости

```bash
# Удалить AUR пакеты которые больше не требуются ничему
yay -Yc

# Или более простой вариант:
yay -c

# yay умнее чем pacman -Qdt
# Проверяет даже сложные зависимости
```

### Очистить кэш пакетов

```bash
# Очистить кэш скачанных пакетов (как pacman -Sc)
yay -Sc

# Удалить всё из кэша (как pacman -Scc)
yay -Scc

# Показать размер кэша
du -sh ~/.cache/yay
```

### Очистить невуспешные сборки

```bash
# Если сборка пакета упала, могут остаться файлы
# Очистить:
rm -rf ~/.cache/yay/package_name

# Или все неудачные:
yay --clean --all
```

---

## ЧАСТЬ 7: Статистика и информация

### Статистика системы

```bash
# Показать статистику системы (количество пакетов и т.д.)
yay -Ps

# Показывает:
# - Всего пакетов
# - Из AUR
# - Размер установки
# - Размер кэша
# - Время на сборку в среднем
```

### Статистика AUR

```bash
# Информация о текущем AUR пакете (нужна папка пакета)
cd ~/.cache/yay/package_name
yay -B

# Или посмотреть на сайте:
# https://aur.archlinux.org/package_name.html
```

---

## ЧАСТЬ 8: Конфигурация yay

### Файл конфигурации

```bash
# yay хранит конфиг здесь:
~/.config/yay/config.json

# Или сгенерировать:
yay -P -g > ~/.config/yay/config.json

# Основные опции:
# - editor: какой редактор использовать (nano, vim)
# - aururl: URL AUR репозитория
# - bottomup: порядок показа результатов
# - compiledir: где собирать пакеты
```

### Полезные опции при запуске

```bash
# Не показывать цвета в выводе
yay --color never

# Использовать конкретный редактор
EDITOR=vim yay package_name

# Не очищать папку после установки (для отладки)
yay --keepbuild package_name

# Использовать конкретный makepkg флаг
yay -S --make-flags="-j4" package_name
# Собирать на 4 ядрах вместо всех
```

---

## ЧАСТЬ 9: Решение проблем

### "error: target not found"

```bash
# Пакет не найден в AUR

# Решение:
# 1. Проверить написание
yay -Ss similar_name

# 2. Может переименовался
yay -Ss new_name

# 3. Пакет удален из AUR
# Используйте альтернативу:
yay -Ss "alternative package"
```

### Ошибки при сборке (error during compilation)

```bash
# Решение (в порядке):

# 1. Обновить систему
sudo pacman -Syu

# 2. Удалить папку пакета и пересобрать
rm -rf ~/.cache/yay/package_name
yay -S package_name

# 3. Проверить что base-devel установлен
pacman -Qi base-devel

# 4. Если совсем плохо - отредактировать PKGBUILD
yay -S package_name
# Нажмите Enter для редактирования PKGBUILD
```

### "failed to create package from PKGBUILD"

```bash
# PKGBUILD содержит ошибки

# Решение:
# 1. Посмотреть что не так
cd ~/.cache/yay/package_name
cat PKGBUILD | grep -A5 "function\|error"

# 2. Попробовать более новую версию
yay -Su package_name

# 3. Если не помогает - report на AUR
# https://aur.archlinux.org/package_name.html
# Кнопка "Flag package out-of-date"
```

### Медленная сборка (долгая компиляция)

```bash
# Пакет долго собирается (часы!)

# Решение (увеличить скорость):

# 1. Собирать на нескольких ядрах
# Отредактировать /etc/makepkg.conf:
sudo nano /etc/makepkg.conf

# Найти:
#MAKEFLAGS="-j2"

# Раскомментировать и изменить на кол-во ядер:
MAKEFLAGS="-j8"  # 8 ядер

# 2. Включить ccache (кэширование компиляции)
sudo pacman -S ccache

# 3. Отключить отладочные символы (экономит время/место)
# В /etc/makepkg.conf измените:
OPTIONS=(!strip docs libtool staticlibs emptydirs !zipman !purge !debug)
```

### Конфликт с официальным пакетом

```bash
# Пытаетесь установить из AUR но есть официальный

# Пример:
yay -S gcc  # В официальных репо уже есть

# Решение:
# 1. Если нужна версия из AUR:
sudo pacman -R gcc  # Удалить официальный
yay -S gcc          # Установить из AUR

# 2. Если нужен официальный:
yay -R gcc-aur      # Удалить AUR версию
sudo pacman -S gcc  # Переустановить официальный

# 3. Обычно официальный лучше (стабильнее)
```

---

## ЧАСТЬ 10: Лучшие практики и советы

### Какие пакеты из AUR выбирать?

```bash
# Хорошие признаки AUR пакета:
# 1. Много голосов (votes > 10)
yay -Si package | grep Votes

# 2. Недавно обновлен
yay -Si package | grep "Last Modified"

# 3. Совместим с последней версией приложения
yay -Si package | grep Version

# 4. Автор известный/надежный
yay -Si package | grep Maintainer
```

### Перед обновлением yay

```bash
# 1. Прочитать новости
# https://aur.archlinux.org/yay.html

# 2. Сделать backup списка пакетов
yay -Qm > ~/packages-aur.txt

# 3. Обновить систему
sudo pacman -Syu

# 4. Обновить yay
yay -S yay
```

### Автоматические обновления (cron)

```bash
# Добавить в crontab:
crontab -e

# Добавить строку (каждый день в 2:00):
0 2 * * * yay -Syu --noconfirm > /tmp/yay-update.log 2>&1

# Или через systemd timer (лучше):
sudo systemctl edit --full yay-update.timer
```

### Использование .SRCINFO для отладки

```bash
# Если пакет не собирается, помогает SRCINFO
cd ~/.cache/yay/package_name

# Посмотреть зависимости в SRCINFO:
grep -E "depends|makedepends" .SRCINFO

# Может помочь понять что не так
```

---

## ЧАСТЬ 11: Шпаргалка (быстрый справочник)

### Основные команды yay

```bash
# УСТАНОВКА
yay package_name                      # Установить из AUR
yay -S package_name                   # Явно (как pacman)
yay -S --noconfirm package_name       # Без вопросов

# УДАЛЕНИЕ
yay -R package_name                   # Удалить
yay -Rs package_name                  # + зависимости
yay -Rns package_name                 # + конфиги

# ОБНОВЛЕНИЕ
yay -Syu                              # Обновить всё
yay -Sua                              # Только AUR пакеты
yay -Qu                               # Показать обновления

# ПОИСК И ИНФОРМАЦИЯ
yay -Ss keyword                       # Поиск везде
yay -Qs keyword                       # Поиск установленных
yay -Si package                       # Инфо из AUR
yay -Qi package                       # Инфо установленного
yay -Ql package                       # Файлы пакета

# AUR УПРАВЛЕНИЕ
yay -Qm                               # Список AUR пакетов
yay -Qn                               # Список официальных
yay -Ps                               # Статистика системы

# ОЧИСТКА
yay -Yc                               # Удалить орфан-пакеты
yay -Sc                               # Очистить кэш
yay -Scc                              # Полная очистка

# КОНФИГУРАЦИЯ
yay -P -g                             # Показать конфиг
yay -P -s                             # Переключить AUR helper
```

### Быстрые сценарии

```bash
# Найти и установить программу
yay -Ss firefox     # Поиск
yay -Si firefox     # Информация
yay firefox         # Установка

# Обновить AUR и очистить
yay -Sua            # Обновить AUR
yay -Yc             # Очистить орфан-пакеты
yay -Sc             # Очистить кэш

# Переустановить AUR пакет (с очисткой)
yay -Rns package    # Удалить полностью
yay package         # Переустановить
```

---

## ЧАСТЬ 12: Альтернативные способы установки из AUR

### Ручная установка (без yay)

```bash
# Если не хотите yay, можно установить вручную:

# 1. Клонировать пакет
git clone https://aur.archlinux.org/package_name.git
cd package_name

# 2. Посмотреть PKGBUILD
cat PKGBUILD

# 3. Собрать
makepkg -si

# Это медленнее но дает полный контроль
```

### Через auracle (легковесный поиск)

```bash
# Установить auracle
sudo pacman -S auracle

# Поиск в AUR
auracle search keyword

# Информация
auracle info package

# Клонировать для установки
auracle clone package
cd package
makepkg -si
```

---

## Связанные заметки

- [[pacman-basic-commands]] - pacman менеджер пакетов
- [[arch-maintenance]] - обслуживание Arch Linux
- [[arch-troubleshooting]] - решение проблем Arch

## Источники

- `yay --help` - встроенная помощь
- AUR Wiki: AUR
- Arch Wiki: AUR helpers
- https://aur.archlinux.org - сайт AUR

---
Создано: 2026-01-01 18:19
